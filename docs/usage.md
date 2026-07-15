# Usage & tools

Once connected, Sankshep exposes these tools to your MCP client's model. In Copilot that means **Agent
mode**; in Claude clients they're available directly.

## Tools

| Tool | What it does |
|---|---|
| **`get_context`** | Token-minimized, ranked context for given paths: strips comments, collapses non-target method bodies, packs under a token budget, and reports the savings. Returns the code as plain, readable text led by a short header stating what was dropped (so nothing is silently truncated), with the savings as structured content. |
| **`search_code`** | Semantic search over a local embedding index (bge-small, ONNX, offline) blended with lexical ranking. |
| **`index_repo`** | Builds/refreshes the semantic index over code **and** `.docx` / `.pdf` documents. |
| **`summarize_repo`** | Tree-sitter-backed API surface of the repository. |
| **`remember` / `recall`** | A per-repo, branch-scoped fact store — decisions and conventions that persist across sessions and clients. |
| **`export_decisions`** | Writes remembered decisions to a `DECISIONS.md`. |
| **`token_report`** | Cumulative tokens (and dollars) saved by minimization, per tool. |

## Prompts

| Prompt | What it does |
|---|---|
| **`compose_task_prompt`** | Assembles a grounded, one-shot prompt for a coding task from minimized code (`get_context`) + recalled conventions (`recall`). A prompt, not an answer. See the [composer guide](composer.md). |

MCP clients that surface prompts (Claude Code, Claude Desktop) expose `compose_task_prompt` as a
**slash-command**.

## A typical flow

1. **Index once:** ask the assistant to run `index_repo` on your repo (or a subtree). This embeds code +
   documents locally.
2. **Ask grounded questions:** the model calls `get_context` / `search_code` to pull only the relevant,
   minimized slice — not whole files.
3. **Remember decisions:** "Remember: we sign JWTs with RS256, keys rotate monthly." The fact is stored
   locally and **recalled later, even from a different client**.
4. **Compose a task:** run `compose_task_prompt` to get a grounded, structured prompt for a change.

## Cross-client memory

Because facts live in a local database every client reads, a decision recorded in Claude Code can be
recalled next week in Copilot Chat — cross-client shared memory is an emergent benefit of keeping
everything local. See [How it works](how-it-works.md).

## Measuring the savings

`token_report` returns cumulative tokens and dollars saved per tool, so "how much did Sankshep save?"
is a number you read, not guess. Methodology: [Benchmarks](benchmarks.md).
