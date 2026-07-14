# Sankshep

*Sankshep* (संक्षेप) is Sanskrit for "the concise essence" — a summary that keeps the substance and
drops the excess.

**Maximum context, minimum tokens — with the benchmarks to prove it.**

Sankshep is a proprietary [MCP](https://modelcontextprotocol.io) server, written in C#/.NET 9, that
gives MCP clients — VS Code Copilot Chat, Claude Code, Claude Desktop, Cursor — **token-minimized,
relevance-ranked codebase context**, plus a **persistent, per-repo memory** of facts and decisions.

Everything runs **locally**: embeddings via ONNX Runtime, vector search via sqlite-vec, **no code or
content leaves your machine** and there is **no telemetry by default**.

## Why it exists

An AI assistant is only as good as the context it's given — but whole files burn tokens and bury the
answer. Sankshep sends the model the *small, relevant slice* instead: it retrieves the chunks that
matter and AST-minimizes them (strips comments, collapses non-target method bodies), then reports how
many tokens it saved.

<div class="grid cards" markdown>

- :material-scissors-cutting: **Minimize** — ranked, comment-stripped, body-collapsed context under a
  token budget ([`get_context`](usage.md)).
- :material-magnify: **Search** — semantic + lexical code search over a local embedding index
  ([`search_code`](usage.md)).
- :material-brain: **Remember** — a per-repo, branch-scoped fact store that survives across sessions and
  clients ([`remember` / `recall`](usage.md)).
- :material-package-variant: **Compose** — a grounded one-shot prompt from minimized code + memory
  ([the prompt composer](composer.md)).

</div>

## Get started

- **[Install](install.md)** — `dotnet tool install -g sankshep`, then point your MCP client at it.
- **[Usage & tools](usage.md)** — the full tool surface.
- **[How it works](how-it-works.md)** — the three-layer model, MCP, and where Sankshep fits.
- **[Deployment](deployment.md)** — Docker, Windows Service, systemd, Kubernetes, air-gap.
- **[Architecture](architecture.md)** — the engineering behind it.

!!! note "Proprietary, free to use"
    Sankshep is proprietary software, **free to install and use** (including commercially). Its source
    is private and it does not accept external pull requests — but the tool itself stays a public,
    free-to-install package. See [Feedback](feedback.md) to reach the maintainer.
