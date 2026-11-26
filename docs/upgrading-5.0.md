# Upgrading to 5.0.0

5.0.0 redesigns the tool surface. **Every tool is renamed, and none of the old names is kept as an alias**, so
a client configuration, prompt file or agent that names a 4.x tool stops working until it is updated. The
server, its transports, its environment variables (one added) and its startup refusals are otherwise as they
were in 4.0.0.

Coming from 3.0.0 or earlier? Read [Upgrading to 4.0.0](upgrading-4.0.md) first; its HTTP-transport refusals
still apply.

The quickest route is `sankshep init` (see [Agents](agents.md)): it writes agents that already use the new
names. Then update anything of your own that names the old ones, using the table below.

---

## 1. The renames

| 4.x | 5.0.0 | What to change |
|---|---|---|
| `get_context(paths, query, tokenBudget, level)` | [`read`](tool-reference.md#read)`(paths, query, tokenBudget, level, symbols, secrets, lineNumbers)` | Rename the call. Lines now carry an `N|` prefix; pass `lineNumbers=false` to drop it. A path may end in `:start-end` |
| `search_code(query, topK)` | [`find`](tool-reference.md#find)`(query, mode, paths, glob, caseSensitive, maxResults, contextLines, secrets)` | Rename the call, and `topK` to `maxResults`. `mode=semantic` is the default and is the old search |
| `summarize_repo(path, maxTokens)` | [`outline`](tool-reference.md#outline)`(path, maxTokens, include)` | Rename the call. `maxTokens` now defaults to 4,000 and bounds the whole answer |
| `index_repo(path, force)` | [`index`](tool-reference.md#index)`(path, force)` | Rename the call |
| `token_report()` | [`report`](tool-reference.md#report)`(window, format)` | Rename the call. Read `format=json` if a program parses it |
| `export_decisions()` | [`recall`](tool-reference.md#recall)`(format=markdown)` | Write the returned `markdown` field to `DECISIONS.md` yourself |
| `compose_task_prompt(task, paths, tokenBudget)` | [`implement`](tool-reference.md#implement)`(task, paths, tokenBudget)` | Rename the prompt |

`remember` and `recall` keep their names and arguments; `recall` gains `format`. The `sankshep://stats` resource
keeps its URI.

## 2. Output you may parse has changed

- **`read`'s header** begins `// ---- Sankshep read`, and every code line carries `N|`. A program that strips the
  header by its old first line, or expects bare code, needs updating, or pass `lineNumbers=false`.
- **The elision marker** is ASCII and cites source lines: `{ /* L18-21 elided */ }`.
- **Balanced keeps fewer bodies on a broad query**: the bodies the query is most about, at most twelve per
  file, instead of every body that mentions any of its words. Name a method in `symbols` to keep its body, or
  read its range.
- **`outline`** leads with the projects-and-manifests inventory, and each declaration ends in its line, `  L11`.
- **`report`** shows every tool's usage, one row per call, with the compression claim on a line of its own.

## 3. Secrets are masked by default

`read` and `find` replace values that look like secrets with `[MASKED len=N]` and list each one in the header by
file, line and key. If your workflow needs the values, for example when troubleshooting a connection string:

- per call, over stdio: `secrets="reveal"`;
- server-wide: set `SANKSHEP_SECRETS=reveal` (see [Environment](environment.md)). Over HTTP a per-call reveal
  is refused unless the server sets this.

## 4. One index rebuild, and the stats move to a new schema

- **`index.db` is rebuilt once** on the first start after upgrading: the index now covers Markdown and manifests,
  and stores masked text only. Allow for one full re-index.
- **`stats.db` moves to schema 3** on first start. 4.x's totals are kept and reported on their own line; they
  are never added to the new figures. `facts.db` is untouched.

## 5. Re-point your client configuration and agents

- A hand-written agent, prompt file or rules file that names a 4.x tool needs the new name. `sankshep init`
  writes current ones and never overwrites a file you edited.
- MCP prompts: `compose_task_prompt` is `implement`, and three prompts are new: `explore`, `architect` and
  `review`.
- Your server registration (`.mcp.json`, `.vscode/mcp.json`, `claude_desktop_config.json`) does not change:
  the command is still `sankshep serve --repo <path>`.

## What is new and needs nothing from you

- `initialize` returns server instructions, which clients such as Claude Code load at session start.
- `sankshep report --repo <path>` prints the report without an MCP client.
- Piping input into a bare `sankshep` prints a hint naming `sankshep serve --repo <path>`.

## What has not changed

- **Environment variable names**, except the one added (`SANKSHEP_SECRETS`).
- **Every 4.0.0 refusal**: the routable-bind `SANKSHEP_ALLOWED_HOSTS` requirement, 32-character API keys, both
  or neither OAuth variables. Exit code 1, one line on stderr.
- **Zero egress by default**, the loopback default bind, .NET 10, and the published platforms (`win-x64`,
  `linux-x64`, `linux-arm64`, `osx-arm64`).

## If something goes wrong

[Troubleshooting](troubleshooting.md) covers the common cases, and [Feedback](feedback.md) is the channel for
anything it does not.
