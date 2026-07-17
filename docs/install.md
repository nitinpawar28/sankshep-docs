# Install

Sankshep is a .NET tool. Install it once, then point any MCP client at it. It speaks MCP over **stdio**
by default (the individual-developer path) and can also serve **Streamable HTTP** for services and
containers (see [Deployment](deployment.md)).

## Install the tool

The RID-specific package bundles the native assets (tree-sitter, sqlite-vec, ONNX Runtime) for your
platform:

```bash
dotnet tool install -g sankshep
sankshep serve --repo .
```

Requires the [.NET SDK/runtime 9](https://dotnet.microsoft.com/download/dotnet/9.0). Update with
`dotnet tool update -g sankshep`.

!!! tip "One-shot with dnx"
    With the .NET 10 SDK you can run the tool without a global install — handy for trying it out or
    pinning a version per project:

    ```bash
    dnx sankshep serve --repo .          # latest
    dnx sankshep@1.4.0 serve --repo .    # pinned version
    ```

## Point your MCP client at it

Sankshep runs as a subprocess of your MCP client; don't run it standalone in a terminal. Configure the
client to launch `sankshep serve --repo <your-repo>`.

=== "VS Code (Copilot Chat, Agent mode)"

    Create `.vscode/mcp.json` in your workspace:

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

    MCP tools only fire in **Agent mode** — pick it in the Chat view, not "Ask".

=== "Claude Code"

    ```bash
    claude mcp add sankshep -- sankshep serve --repo .
    ```

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

Omit `headers` if the server runs in `None` auth mode on loopback. To accept non-loopback hosts you must set
**both** `ASPNETCORE_URLS` and `SANKSHEP_ALLOWED_HOSTS` — see [Security & privacy](security.md).

## Environment variables

All optional. Defaults keep Sankshep local-only with no telemetry.

| Variable | Purpose | Default |
|---|---|---|
| `SANKSHEP_STATE_DIR` | Where per-repo state (`facts/index/stats.db`) is written | `<repo>/.sankshep` |
| `SANKSHEP_MODEL_DIR` | Embedding-model cache location | per-user default |
| `SANKSHEP_MODEL_OFFLINE` | `1` = never attempt the model download (air-gap; side-load first) | off |
| `SANKSHEP_WATCH` | `1` = watch the working tree and keep the index fresh automatically | off |
| `SANKSHEP_DISABLE_VEC` | `1` = disable the sqlite-vec native path (pure-C# vector fallback) | off |
| `SANKSHEP_OTLP_ENDPOINT` | Enable OTLP metric export to this collector (opt-in) | unset (no export) |
| `SANKSHEP_PROMETHEUS` | `1` = expose the `/metrics` scrape endpoint (opt-in) | off |
| `SANKSHEP_FLEET_TEAM` / `SANKSHEP_FLEET_INSTANCE` | Optional low-cardinality labels on exported metrics | unset |
| `ASPNETCORE_URLS` | HTTP bind address(es) | `http://127.0.0.1:8080` |
| `SANKSHEP_ALLOWED_HOSTS` | Host-header allowlist required to accept non-loopback traffic | loopback only |
| `SANKSHEP_API_KEY` / `SANKSHEP_API_KEYS` | Bearer key(s) for ApiKey auth mode (HTTP) | unset (no key auth) |
| `SANKSHEP_OAUTH_AUTHORITY` / `_AUDIENCE` / `_RESOURCE` / `_SCOPES` | OAuth 2.1 resource-server validation (HTTP) | unset (OAuth off) |

## First run

The first semantic search or index builds a local embedding index and, once, downloads the embedding
model (~127 MB) to `~/.sankshep/models`. After that, everything is offline. In air-gapped environments
you can side-load the model and set `SANKSHEP_MODEL_OFFLINE=1` — see [Deployment](deployment.md).

Next: the [tool reference](tool-reference.md), a [5-minute quickstart](quickstart.md), and the
[prompt composer](composer.md).
