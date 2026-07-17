<p class="sankshep-hero"><img src="assets/logomark.png" alt="Sankshep"></p>

# Sankshep

*Sankshep* (संक्षेप) is Sanskrit for "the concise essence" — a summary that keeps the substance and
drops the excess.

**Maximum context, minimum tokens — with the tooling to measure it on your own code.**

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

The full tool surface — nine primitives, each documented with real captured I/O in the
[tool reference](tool-reference.md):

<div class="grid cards" markdown>

- :material-scissors-cutting: **Minimize** — ranked, comment-stripped, body-collapsed context under a
  token budget ([`get_context`](tool-reference.md#get_context)).
- :material-magnify: **Search** — semantic + lexical search over a local embedding index you build with
  `index_repo` ([`search_code`](tool-reference.md#search_code)).
- :material-file-tree: **Map** — namespaces, public types, and method signatures for a directory
  ([`summarize_repo`](tool-reference.md#summarize_repo)).
- :material-brain: **Remember** — a per-repo, branch-scoped fact store that survives across sessions and
  clients ([`remember` / `recall`](tool-reference.md#remember)).
- :material-file-document-check: **Decisions** — export decision-tagged memory to `DECISIONS.md`
  ([`export_decisions`](tool-reference.md#export_decisions)).
- :material-counter: **Measure** — cumulative token compression, per tool, no dollar guesswork
  ([`token_report`](tool-reference.md#token_report)).
- :material-package-variant: **Compose** — a grounded one-shot prompt from minimized code + memory
  ([the prompt composer](composer.md)).

</div>

## Get started

- **[Quickstart](quickstart.md)** — install → connect → first grounded answer → see the savings, in ~5 minutes.
- **[Install](install.md)** — `dotnet tool install -g sankshep`, client configs, HTTP, and environment variables.
- **[Tool reference](tool-reference.md)** — every tool's arguments and a real captured request/response.
- **[How it works](how-it-works.md)** — the three-layer model, MCP, and where Sankshep fits.
- **[Deployment](deployment.md)** — Docker, Windows Service, systemd, Kubernetes, air-gap.
- **[Troubleshooting](troubleshooting.md)** · **[Security & privacy](security.md)** · **[Architecture](architecture.md)** — when something misbehaves, what stays local, and the engineering behind it.

!!! note "Proprietary, free to use"
    Sankshep is proprietary software, **free to install and use** (including commercially). Its source
    is private and it does not accept external pull requests — but the tool itself stays a public,
    free-to-install package. See [Feedback](feedback.md) to reach the maintainer.
