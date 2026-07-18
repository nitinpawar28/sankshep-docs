# Usage & tools

Once connected, Sankshep exposes these tools to your MCP client's model. In Copilot that means **Agent
mode**; in Claude clients they're available directly.

## Tools

| Tool | What it does |
|---|---|
| **`get_context`** | Token-minimized context for given paths, relevance-ranked (semantic + lexical): strips comments, collapses method bodies that aren't relevant to your query, packs under a token budget, and reports the savings. Returns everything as one plain, readable text block — a short header stating how much was compressed and what was withheld (so nothing is silently truncated), followed by the code. |
| **`search_code`** | Semantic (nearest-neighbor) search over a local embedding index (bge-small, ONNX, offline). |
| **`index_repo`** | Builds/refreshes the semantic index over code **and** `.docx` / `.pdf` documents. |
| **`summarize_repo`** | Tree-sitter-backed API surface of a repository's C# (`.cs`) files. |
| **`remember` / `recall`** | A per-repo, branch-scoped fact store — decisions and conventions that persist across sessions and clients. |
| **`export_decisions`** | Writes remembered decisions to a `DECISIONS.md`. |
| **`token_report`** | Cumulative token accounting per tool: how far minimization compressed the code it delivered, and how much context it searched to find it. |

## Prompts

| Prompt | What it does |
|---|---|
| **`compose_task_prompt`** | Assembles a grounded, one-shot prompt for a coding task from minimized code (`get_context`) + recalled conventions (`recall`). A prompt, not an answer. See the [composer guide](composer.md). |

MCP clients that surface prompts (Claude Code, Claude Desktop) expose `compose_task_prompt` as a
**slash-command**.

## Supported languages

Sankshep parses code with tree-sitter. Every language below is pinned by a golden test — Sankshep claims
only what it verifies.

| Language | Comment strip | Body collapse | Notes |
|---|---|:---:|---|
| C# | ✓ | ✓ | Trims unused `using`s at `level=Aggressive`. |
| JavaScript / TypeScript | ✓ | ✓ | `.js .mjs .cjs .jsx .ts .tsx` |
| Go | ✓ | ✓ | |
| Java | ✓ | ✓ | |
| C / C++ | ✓ | ✓ | `.c .h` → C; `.cpp .cc .cxx .hpp .hh .hxx` → C++ |
| Rust | ✓ | ✓ | |
| PHP | ✓ | ✓ | |
| Python | ✓ | — | Indentation blocks have no brace body to elide. |
| Ruby | ✓ | — | `def…end` blocks have no brace body to elide. |

**Body collapse** replaces a method/function body that isn't relevant to your query with a
`{ /* … body elided (N lines) */ }` marker, so it applies only to brace-bodied languages; Python and Ruby
get comment-stripping and packing but keep their bodies. Files in **any other** language are still
retrieved, ranked, and packed under your token budget — just at the **text level**, without parse-aware
minimization.

!!! note "Minimization levels"
    `get_context` takes a `level` argument (see the [tool reference](tool-reference.md#get_context)).
    **Conservative** collapses nothing; **Balanced** (the default) keeps the method bodies relevant to your
    query and collapses the rest; **Aggressive** collapses every non-relevant body plus expression-bodied
    (`=>`) members and trims unused `using`s.

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

`token_report` returns cumulative token accounting per tool: **compression** — delivered tokens against
the original size of the same files, i.e. what minimization actually removed — plus how much context was
**searched** to find them.

Searched tokens are reported for context and are deliberately **not** a baseline: Sankshep may read a whole
repository to select a handful of files, and that is not context you would otherwise have sent. Counting it
as a saving is how a tool ends up claiming it saved more tokens than the model could ever accept. The value
of *selecting* the right files is a retrieval question, and recall — not a token count — is what answers it.

There is no dollar figure. Sankshep is never told which model you use or what you pay for it, so any total
would be an assumption dressed as a measurement. Multiply compressed tokens by your own negotiated input
rate if you want one. Methodology: [Benchmarks](benchmarks.md).
