# Usage & tools

Once connected, Sankshep exposes these tools to your MCP client's model. In Copilot that means **Agent
mode**; in Claude clients they're available directly.

## Tools

| Tool | What it does |
|---|---|
| **`read`** | Token-minimized context for given paths, relevance-ranked (semantic + lexical): strips comments, collapses method bodies that aren't relevant to your query, packs under a token budget, and reports the savings. Every line is prefixed `N|` with its source line number, so an answer can cite `path:N` (`lineNumbers=false` drops it). A path ending in `:start-end` returns those exact lines, unminimized; `symbols` keeps the named bodies whole; a Markdown, manifest or configuration file you name comes back as written. When a file does not fit what is left of the budget, its most relevant declarations come back instead, each cited `// path:start-end Symbol`, and the header's `PARTIAL` line names the file (4.0.0). Secret values are masked as `[MASKED len=N]` unless you pass `secrets="reveal"` (5.0.0). Returns everything as one plain, readable text block — a short header stating how much was compressed and what was withheld (so nothing is silently truncated), followed by the code. |
| **`find`** | Finds code and text, each hit numbered `N|` and masked like `read`. `mode=exact` and `mode=regex` scan the working tree as it is now — use them instead of grep — and `mode=symbol` finds where a type or method is declared; none of the three needs an index. `mode=semantic`, the default, is nearest-neighbor search over a local embedding index (bge-small, ONNX, offline). `maxResults` defaults to 20: up to 100 for `semantic`, 500 for the other modes. |
| **`index`** | Builds/refreshes the semantic index over code, Markdown, manifests **and** `.docx` / `.pdf` documents. Secrets are masked before anything is stored. |
| **`outline`** | An inventory of the projects, packages, scripts, container images and pipelines under a directory, each cited by line, then the tree-sitter-backed declarations in its source files, each with its line, in every language Sankshep parses. `maxTokens` (default 4,000) is a ceiling on the whole answer, and it says so when it cuts. |
| **`remember` / `recall`** | A per-repo, branch-scoped fact store — decisions and conventions that persist across sessions and clients. Facts are at most 4,000 characters, categories at most 64 (4.0.0). `recall` with `format=markdown` returns the decisions as `DECISIONS.md` text, in a `markdown` field, for you to write where you want it; nothing is written to your repository. |
| **`report`** | Every tool and prompt call, one row per call, with its outcome, time and the tokens it returned, for this session, the last 24 hours, the last 7 days or every call kept (`window`; the newest 100,000 are kept), as text or JSON (`format`). How far `read` compressed the code it delivered is reported on its own line, apart from that count. |

## Prompts

| Prompt | What it does |
|---|---|
| **`implement`** | Assembles a grounded, one-shot prompt for a coding task from minimized code (`read`) + recalled conventions (`recall`), and lists anything it could not include. A prompt, not an answer. See the [composer guide](composer.md). |
| **`explore`** | Starts the Explorer workflow: answer a `question` about the repository from its code, every claim cited as `path:N`. Changes nothing. |
| **`architect`** | Starts the Architect workflow: map and assess the architecture of the repository, or of the `scope` you name. Changes nothing. |
| **`review`** | Starts the Reviewer workflow: review the change against `base` (default `HEAD`, the uncommitted work) for bugs, broken conventions and missed callers. Changes nothing. |

MCP clients that surface prompts (Claude Code, Claude Desktop) expose them as **slash-commands**. The same
four workflows can be installed as agents with `sankshep init` — see [Agents](agents.md).

## Supported languages

Sankshep parses code with tree-sitter. Every language below is pinned by a golden test — Sankshep claims
only what it verifies.

| Language | Comment strip | Body collapse | Notes |
|---|---|:---:|---|
| C# | ✓ | ✓ | Trims unused `using`s at `level=Aggressive`. |
| JavaScript / TypeScript | ✓ | ✓ | `.js .mjs .cjs .jsx .ts`; `.tsx` uses its own grammar, so JSX inside a component is parsed rather than guessed at |
| Go | ✓ | ✓ | |
| Java | ✓ | ✓ | |
| C / C++ | ✓ | ✓ | `.c` → C; `.cpp .cc .cxx .h .hpp .hh .hxx` → C++ |
| Rust | ✓ | ✓ | |
| PHP | ✓ | ✓ | |
| Python | ✓ | — | Indentation blocks have no brace body to elide. Locators and truncation markers are `#` comments (4.0.0). |
| Ruby | ✓ | — | `def…end` blocks have no brace body to elide. Locators and truncation markers are `#` comments (4.0.0). |

**Body collapse** replaces a method/function body that isn't relevant to your query with a
`{ /* L12-40 elided */ }` marker citing the source lines it stands for (5.0.0), so it applies only to
brace-bodied languages; Python and Ruby get comment-stripping and packing but keep their bodies. Since 5.0.0
any other text file you name in `read`, whether Markdown, a manifest, configuration or source in a language
not on this table, comes back as written, numbered by line. Such files are not minimized, and `read` never
picks them up from a directory. A named file is reported on the header's `WARNING: requested but NOT
included` line only when it cannot be returned, for example a binary file or one over the size cap. `index` also indexes Markdown, manifests and `.docx` and `.pdf`
documents for semantic `find`.

!!! note "Minimization levels"
    `read` takes a `level` argument (see the [tool reference](tool-reference.md#read)).
    **Conservative** collapses nothing; **Balanced** (the default) keeps the method bodies your query is
    most about — at most twelve per file since 5.0.0 — and collapses the rest; **Aggressive** collapses every
    body, the ones relevant to your query included, plus expression-bodied (`=>`) members, and trims unused
    `using`s. At any level, naming a method in `symbols` keeps its body whole.

## A typical flow

1. **Index once:** ask the assistant to run `index` on your repo (or a subtree). This embeds code +
   documents locally, for semantic `find`.
2. **Ask grounded questions:** the model calls `outline` to orient, `find` to locate and `read` to pull
   only the relevant, minimized slice — not whole files — and cites `path:N` from the line numbers.
3. **Remember what you decided:** "Remember: we sign JWTs with RS256, keys rotate monthly." The fact is
   stored locally and **recalled later, even from a different client**.
4. **Compose a task:** run `implement` to get a grounded, structured prompt for a change.

!!! tip "Category matters for step 4"
    `implement`'s "Project conventions" section injects facts remembered under the
    **`convention`** category and no other. File a rule you want the model to follow as `convention`;
    `decision` is for the record and for `recall format=markdown`. The composed prompt names the categories
    it skipped, so an empty section tells you where your facts went.

## Cross-client memory

Because facts live in a local database every client reads, a decision recorded in Claude Code can be
recalled next week in Copilot Chat — cross-client shared memory is an emergent benefit of keeping
everything local. See [How it works](how-it-works.md).

## Measuring the savings

`report` counts every tool and prompt call and the tokens each returned — what those calls added to the
caller's context — and states **compression** on a line of its own: delivered tokens against the original
size of the source that came back (whole files, or the declarations delivered from a file that did not
fit), i.e. what minimization actually removed. `sankshep report --repo <path>` prints the report from a
terminal, with no MCP client, from a copy of `stats.db`, so nothing in the state directory changes. It defaults to
`--window all`, where the tool defaults to `session`. `stats.db` keeps the newest 100,000 calls, so `all` means
all that are kept.

Context **searched** is deliberately **not** a baseline: Sankshep may read a whole
repository to select a handful of files, and that is not context you would otherwise have sent. Counting it
as a saving is how a tool ends up claiming it saved more tokens than the model could ever accept. The value
of *selecting* the right files is a retrieval question, and recall — not a token count — is what answers it.

There is no dollar figure. Sankshep is never told which model you use or what you pay for it, so any total
would be an assumption dressed as a measurement. Multiply compressed tokens by your own negotiated input
rate if you want one. Methodology: [Benchmarks](benchmarks.md).
