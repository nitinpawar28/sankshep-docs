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
    If you have the .NET 10 SDK, you can run the tool without a global install using `dnx` — handy for
    trying it out or pinning a version per project.

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

## First run

The first semantic search or index builds a local embedding index and, once, downloads the embedding
model (~127 MB) to `~/.sankshep/models`. After that, everything is offline. In air-gapped environments
you can side-load the model and set `SANKSHEP_MODEL_OFFLINE=1` — see [Deployment](deployment.md).

Next: the [tool surface](usage.md) and the [prompt composer](composer.md).
