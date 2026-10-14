# Install

Sankshep is a .NET tool. Install it once, then point any MCP client at it. It speaks MCP over **stdio**
by default (the individual-developer path) and can also serve **Streamable HTTP** for services and
containers (see [Deployment](deployment.md)).

## Install the tool

The RID-specific package bundles the native assets (tree-sitter, sqlite-vec, ONNX Runtime) for your
platform. Supported platforms are **win-x64**, **linux-x64**, **linux-arm64** and **osx-arm64**;
`win-arm64` is not packaged, because there is no sqlite-vec native for it, and a platform that is not
packaged fails cleanly at install rather than at runtime:

!!! warning "Intel Macs (`osx-x64`): search and indexing cannot work"

    **`sankshep.osx-x64` is not published from 3.0.0 onwards.** It existed up to 2.0.0 and should not
    have: **Microsoft.ML.OnnxRuntime ships no `osx-x64` native**, so the package installed, ran and
    answered `--version` — and then failed the first time anything needed an embedding. Over stdio
    `index_repo` and `search_code` failed while the other six tools worked; over HTTP the server never
    became ready and answered 503 for the life of the process. It is the one platform where "fails
    cleanly at install" was never true, which is why withdrawing it is breaking rather than a cleanup.

    **An Intel Mac on 2.0.0 keeps working exactly as it did** — those packages stay on nuget.org,
    because published packages are immutable — but cannot install or update to 3.0.0 or later. **The
    container image runs fully on an Intel Mac**, since it is published for `linux/amd64`; it serves the
    HTTP transport, and the `docker run` recipe on [Deployment](deployment.md#docker) sets the key and the
    `Host` allow-list 4.0.0 requires. Apple Silicon is unaffected: use `osx-arm64`.

```bash
dotnet tool install -g sankshep
sankshep --version                   # verify the install (serve is launched by your MCP client)
```

Requires the [.NET 10 SDK](https://dotnet.microsoft.com/download/dotnet/10.0), **or** the .NET 10
runtime plus the **ASP.NET Core 10 shared runtime** — the server framework-references
`Microsoft.AspNetCore.App` even in stdio mode, so the plain .NET runtime on its own is not enough. The SDK
includes both. Update with `dotnet tool update -g sankshep`.

!!! tip "One-shot with dnx"
    With the .NET 10 SDK you can run the tool without a global install — handy for trying it out or
    pinning a version per project:

    ```bash
    dnx sankshep serve --repo .          # latest
    dnx sankshep@4.0.0 serve --repo .    # pinned version
    ```

## Point your MCP client at it

Sankshep runs as a subprocess of your MCP client; don't run it standalone in a terminal. Configure the
client to launch `sankshep serve --repo <your-repo>`.

=== "VS Code (Copilot Chat)"

    Create `.mcp.json` at the root of your workspace. This is the **portable** format, with servers under
    a top-level `mcpServers` key: VS Code prefers it for new servers, and other MCP clients read the same
    file — Claude Code among them.

    ```json
    {
      "mcpServers": {
        "sankshep": {
          "command": "sankshep",
          "args": ["serve", "--repo", "."]
        }
      }
    }
    ```

    VS Code starts a stdio server in the workspace folder, so `.` is the repository you opened. (The
    portable file passes `.` rather than VS Code's `${workspaceFolder}`, which Claude Code would pass to
    the server unexpanded.)

    The older `.vscode/mcp.json` form, with servers under `servers`, still works; VS Code now lists it as
    deprecated:

    ```json
    {
      "servers": {
        "sankshep": {
          "command": "sankshep",
          "args": ["serve", "--repo", "${workspaceFolder}"]
        }
      }
    }
    ```

    MCP tools only fire when the chat is running an agent: in the chat input, select **Agent** from the
    agent picker, not Ask.

=== "Claude Code"

    ```bash
    claude mcp add sankshep -- sankshep serve --repo .
    ```

    That adds the server at Claude Code's default **local** scope: it is stored in your own
    `~/.claude.json`, for this project only. To share it with everyone who works on the repository, add it
    at **project** scope instead, which writes it to a `.mcp.json` at the project root for you to commit —
    the same portable file the VS Code tab shows:

    ```bash
    claude mcp add --scope project sankshep -- sankshep serve --repo .
    ```

    In an interactive session, Claude Code asks you to approve a project-scoped server before using it.
    Sankshep's tools and its `compose_task_prompt` slash-command then appear in the session.

=== "Claude Desktop / Cursor"

    Add to the client's MCP config (`claude_desktop_config.json` / `.cursor/mcp.json`):

    ```json
    {
      "mcpServers": {
        "sankshep": {
          "command": "sankshep",
          "args": ["serve", "--repo", "/absolute/path/to/repo"]
        }
      }
    }
    ```

## Connect over HTTP (remote clients)

For a shared or containerized server, run Sankshep in Streamable-HTTP mode and point clients at the URL
instead of launching a subprocess. Start the server (see [Deployment](deployment.md) for service/container
setup):

```bash
sankshep --http --repo /path/to/repo          # binds http://127.0.0.1:8080 (loopback)
```

Then configure the client with a URL transport (and a bearer token if you enabled auth):

```json
{
  "servers": {
    "sankshep": {
      "type": "http",
      "url": "http://127.0.0.1:8080/",
      "headers": { "Authorization": "Bearer ${env:SANKSHEP_TOKEN}" }
    }
  }
}
```

Omit `headers` if the server runs in `None` auth mode on loopback. A non-loopback bind (`ASPNETCORE_URLS` set
to a non-loopback address) **refuses to start** unless you configure both of these: an authentication choice
— `SANKSHEP_API_KEYS` or `SANKSHEP_OAUTH_*`, or, on a trusted network-isolated host only,
`SANKSHEP_ALLOW_UNAUTHENTICATED=1` — and, since 4.0.0, `SANKSHEP_ALLOWED_HOSTS` naming the hosts clients use
(or `*` to accept any `Host` deliberately). Every API key must be at least 32 characters
(`openssl rand -hex 32`). The allow-list does not satisfy the auth gate, and the auth gate does not satisfy
the allow-list — see [Security & privacy](security.md) and [Troubleshooting](troubleshooting.md).

## Environment variables

All optional on the stdio and loopback paths; a routable `--http` bind needs an auth choice and a `Host`
allow-list. Defaults keep Sankshep local-only with no telemetry. See
[Environment variables](environment.md) for the full reference with grouping and notes.

| Variable | Purpose | Default |
|---|---|---|
| `SANKSHEP_STATE_DIR` | Where per-repo state (`facts/index/stats.db`) is written | `<repo>/.sankshep` |
| `SANKSHEP_MODEL_DIR` | Embedding-model cache location | per-user default |
| `SANKSHEP_MODEL_OFFLINE` | `1` = never attempt the model download (air-gap; side-load first) | off |
| `SANKSHEP_WATCH` | `1` = watch the working tree and keep the index fresh automatically | off |
| `SANKSHEP_DISABLE_VEC0` | `1` = disable the sqlite-vec native path (pure-C# vector fallback) | off |
| `SANKSHEP_OTLP_ENDPOINT` | Enable OTLP metric export to this collector (opt-in) | unset (no export) |
| `SANKSHEP_PROMETHEUS` | `1` = expose the `/metrics` scrape endpoint (opt-in) | off |
| `SANKSHEP_FLEET_TEAM` / `SANKSHEP_FLEET_INSTANCE` | Optional low-cardinality labels on exported metrics | unset |
| `ASPNETCORE_URLS` | HTTP bind address(es) | `http://127.0.0.1:8080` |
| `SANKSHEP_ALLOWED_HOSTS` | `Host`-header allow-list, compared whole with the port ignored; `*` accepts any `Host` deliberately and cannot be combined with names. Enforced in the pipeline, so health probes stay reachable whatever it says. | loopback names on a loopback bind; **required on any other bind** — unset or blank refuses to start (4.0.0) |
| `SANKSHEP_ALLOWED_ORIGINS` | `Origin` allow-list for browser callers. A request with no `Origin` is unaffected. | empty (no cross-origin request accepted) |
| `SANKSHEP_ALLOW_UNAUTHENTICATED` | `1` = explicitly permit an unauthenticated non-loopback bind (trusted, network-isolated hosts only) | off — server fails closed on a non-loopback bind with no auth |
| `SANKSHEP_API_KEY` / `SANKSHEP_API_KEYS` | Bearer key(s) for ApiKey auth mode (HTTP); each at least 32 characters, or the server refuses to start (4.0.0) | unset (no key auth) |
| `SANKSHEP_OAUTH_AUTHORITY` / `_AUDIENCE` / `_RESOURCE` / `_SCOPES` | OAuth 2.1 resource-server validation (HTTP). `_AUTHORITY` and `_AUDIENCE` together or not at all; one alone refuses to start (4.0.0) | unset (OAuth off) |

## First run

The first semantic search or index builds a local embedding index and, once, downloads the embedding
model (~127 MiB) to `~/.sankshep/models`. After that, everything is offline. In air-gapped environments
you can side-load the model and set `SANKSHEP_MODEL_OFFLINE=1` — see [Deployment](deployment.md#air-gapped-zero-egress).

Next: the [tool reference](tool-reference.md), a [5-minute quickstart](quickstart.md), and the
[prompt composer](composer.md).
