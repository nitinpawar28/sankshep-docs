# Agents: `sankshep init`

Sankshep ships four agents, and `sankshep init` writes them into your repository for the AI clients you use.
Each one reads and searches through Sankshep (`outline`, `find`, `read`) instead of shell commands and the
client's own file tools, and cites every claim as `path:N`.

| Agent | What it does | Besides Sankshep, it may |
|---|---|---|
| `sankshep-explorer` | Answers questions about the repository from its code | nothing |
| `sankshep-architect` | Maps and assesses the architecture, or one part of it: layers, dependencies, external systems, risks | nothing |
| `sankshep-implementer` | Makes a change: reads exactly the lines it edits, follows the recorded conventions, builds and tests | edit files, run a terminal |
| `sankshep-reviewer` | Reviews a change against a base commit for bugs, broken conventions and missed callers | run a terminal, for `git diff` |

The same four workflows are also MCP prompts, `explore`, `architect`, `implement` and `review` (see the
[Tool reference](tool-reference.md#prompts)), for clients that offer prompts.

## Run it

From the root of your repository, after [installing](install.md) Sankshep:

```bash
sankshep init --dry-run   # show what would be written
sankshep init             # write it
```

Captured from the published 5.0.0 in a repository with none of these files yet:

```text
created .github/agents/sankshep-explorer.agent.md
created .github/agents/sankshep-architect.agent.md
created .github/agents/sankshep-implementer.agent.md
created .github/agents/sankshep-reviewer.agent.md
created .claude/agents/sankshep-explorer.md
created .claude/agents/sankshep-architect.md
created .claude/agents/sankshep-implementer.md
created .claude/agents/sankshep-reviewer.md
created .cursor/rules/sankshep.mdc
created AGENTS.md
created .mcp.json
created .cursor/mcp.json
sankshep init: done.
```

Commit what it wrote, so everyone working on the repository gets the same agents.

| Option | |
|---|---|
| `--repo <path>` | The repository to write into. Default: the current directory |
| `--hosts vscode,claude,agents,cursor` | Which clients to write for, comma-separated, or `all` (the default) |
| `--no-mcp` | Do not touch `.mcp.json` or `.cursor/mcp.json` |
| `--force` | Overwrite generated files you have edited |
| `--dry-run` | Print what would change and write nothing |

## What it writes, per client

| `--hosts` | Files | Notes |
|---|---|---|
| `vscode` | `.github/agents/sankshep-<role>.agent.md` | VS Code Copilot custom agents. Visual Studio 18.4 and later reads the same directory |
| `claude` | `.claude/agents/sankshep-<role>.md` | Claude Code subagents. Cursor reads this directory too |
| `cursor` | `.cursor/rules/sankshep.mdc`, `.cursor/mcp.json` | An always-applied rule with the usage guidance |
| `agents` | One section of `AGENTS.md` | For any client that reads `AGENTS.md` |

Unless you pass `--no-mcp`, it also registers the server in `.mcp.json` at the repository root, which VS Code
Copilot, Claude Code and Visual Studio all read:

```json
{
  "mcpServers": {
    "sankshep": {
      "command": "sankshep",
      "args": [
        "serve",
        "--repo",
        "."
      ]
    }
  }
}
```

The agents expect the server to be registered under the name `sankshep`.

### Which tools each agent gets

The agent files restrict tools wherever the client supports it. That restriction matters: an agent that can
run a terminal and has no file-read tool tends to read files through the terminal, which costs more and cannot
be cited by line.

=== "VS Code Copilot"

    ```yaml
    # sankshep-explorer, sankshep-architect
    tools: ['sankshep/*', 'todo']
    # sankshep-implementer
    tools: ['sankshep/*', 'edit', 'execute', 'todo']
    # sankshep-reviewer
    tools: ['sankshep/*', 'execute', 'todo']
    ```

    The built-in `read` and `search` tools are left out on purpose.

=== "Claude Code"

    ```yaml
    # sankshep-explorer, sankshep-architect
    tools: mcp__sankshep__outline, mcp__sankshep__find, mcp__sankshep__read, mcp__sankshep__index, mcp__sankshep__remember, mcp__sankshep__recall, mcp__sankshep__report
    # sankshep-implementer: the same, plus
    Read, Edit, Write, Bash
    # sankshep-reviewer: the same, plus
    Bash
    ```

    The implementer keeps Claude Code's `Read` because its `Edit` refuses a file `Read` has not opened.

=== "Cursor"

    Cursor's subagents have no tool restriction of their own; they inherit every tool. Cursor reads the
    `.claude/agents/` files, and the `.mdc` rule tells it to prefer Sankshep.

!!! note "Not yet confirmed in Visual Studio"
    Visual Studio reads `.github/agents/` but names its built-in tools differently from VS Code. Whether it
    honours, ignores or rejects the VS Code tool names has not been confirmed. If the agents do not load
    there, run `sankshep init --hosts claude,agents,cursor` and tell us in an
    [issue](https://github.com/nitinpawar28/sankshep-docs/issues).

## It never overwrites your edits

Every generated file carries a line like this inside its front matter:

```yaml
# generated by sankshep 5.0.0 sha256:c642af0451323d611f31f501bb1379ae7000dc89a327cbb98b0f0c8acc527a94
```

The hash is of the rest of the file: everything but that line, compared with LF line endings, so a checkout that
converts line endings does not count as an edit. (In `AGENTS.md` it covers the Sankshep section only.) On the
next run, `init` re-renders a file only while the hash still matches, which means nobody has changed it. A file you edited, or one Sankshep never wrote, is **skipped and named**,
and `init` exits `3`:

```text
skipped .github/agents/sankshep-explorer.agent.md - edited by hand (or not written by sankshep); --force overwrites it
sankshep init: 1 file(s) skipped; the rest was written.
```

Edit the agents freely; upgrading Sankshep and re-running `init` will not undo it. `--force` replaces edited
files with the current version.

- **`AGENTS.md`** is yours. Only the section between `<!-- sankshep:begin ... -->` and `<!-- sankshep:end -->` is
  written, in the file's own line endings. With no section yet, one is appended.
- **`.mcp.json`** gets a `sankshep` entry only when it has none. An existing entry is never changed, and
  your other servers are kept. A file with comments in it is skipped rather than rewritten without them, and
  `init` prints the entry to add by hand.
- **Links are refused.** If any target resolves outside the repository, or through a symbolic link or
  junction, nothing at all is written and `init` exits `1`.

| Exit code | Meaning |
|---|---|
| `0` | Everything is in place |
| `3` | A file was skipped (edited, or could not be merged); everything else was written |
| `1` | Bad arguments, or a refused target, and nothing was written; or a write failed, and the message names the file |

## Keeping it current

Re-run `sankshep init` after upgrading Sankshep. Files you have not touched move to the new version, and
everything else is reported as `unchanged` or `skipped`.
