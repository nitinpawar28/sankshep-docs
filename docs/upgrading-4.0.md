# Upgrading to 4.0.0

Everything 4.0.0 asks of you, in the order it is likely to bite. Over stdio nothing refuses to start, no
index rebuilds, and a `get_context` call with room to spare returns exactly what it did. What can bite is on
the HTTP transport, and in what a call that fills its budget returns.

Coming from 2.0.0 or earlier? Read [Upgrading to 3.0.0](upgrading-3.0.md) first (and from 1.x,
[Upgrading to 2.0.0](upgrading-2.0.md)); their index rebuilds still apply, and this release adds none.

---

## 1. A routable `--http` bind needs `SANKSHEP_ALLOWED_HOSTS`

**Who it affects:** `docker run`, compose, the Windows Service or systemd with a non-loopback
`ASPNETCORE_URLS`, or any other non-loopback bind. Helm users need do nothing: the chart sets the list, and
`allowedHosts.enabled: false` now renders `*`.
**What you do:** set it to the host names clients send (comma-separated; a port is ignored), or to `*` to
accept any `Host` deliberately — `Origin` validation stays on either way.

`*` cannot be combined with names, and a value that names no host (`,`) counts as unset. Without it the
server prints one line on stderr and exits 1:

```text
Refusing to start: the server binds a non-loopback address and SANKSHEP_ALLOWED_HOSTS is not set, so any Host header would be accepted and the DNS-rebinding defence would be off. Set it to the host names clients use to reach this server (comma-separated, e.g. "sankshep.example.com,sankshep"), or to "*" to accept any Host deliberately.
```

!!! info "Why it changed"
    With the variable unset, a routable bind accepted **any** `Host` — the DNS-rebinding defence was off on
    exactly the binds where DNS rebinding matters. Loopback keeps its built-in list and is unaffected.

---

## 2. API keys shorter than 32 characters refuse to start

**Who it affects:** anyone setting `SANKSHEP_API_KEYS` or `SANKSHEP_API_KEY` on any `--http` start,
loopback included; Helm `auth.apiKeys` / `auth.existingSecret` (a short key crash-loops the pod); Windows
Service and systemd installs that pass a key.
**What you do:** generate a key with `openssl rand -hex 32`, then re-issue it to your clients.

The refusal names the variable and how many keys are short, never a key:

```text
Refusing to start: 1 of 1 API key(s) in SANKSHEP_API_KEYS is shorter than 32 characters. A key must be at least 32 characters - generate one with `openssl rand -hex 32`.
```

---

## 3. OAuth needs both `SANKSHEP_OAUTH_AUTHORITY` and `SANKSHEP_OAUTH_AUDIENCE`, or neither

**Who it affects:** OAuth resource-server deployments.
**What you do:** set both, or unset both.

```text
Refusing to start: SANKSHEP_OAUTH_AUTHORITY is set but SANKSHEP_OAUTH_AUDIENCE is not. OAuth needs both SANKSHEP_OAUTH_AUTHORITY and SANKSHEP_OAUTH_AUDIENCE; with one of them the server would fall back to API-key mode or to no authentication at all. Set the other, or unset both.
```

!!! info "Why it changed"
    One variable alone used to resolve silently to API-key mode or to no authentication, so a server its
    operator believed was behind an identity provider was not.

---

## 4. `get_context` on calls that fill `tokenBudget`

**Who it affects:** scripts that parse the header or the locators, and anyone comparing compression
figures across versions.
**What you do:** read counts, not names; check the `PARTIAL` line; raise `tokenBudget` when you need the
full lists.

What changes, in the order you will see it:

- **A file that does not fit what is left of the budget arrives by its declarations**, the ones relevant to
  the query first, each under `// path:start-end Symbol`. Contiguous pieces share one locator without a
  symbol name. A file that fits whole is unchanged.
- **A new header line**, `// PARTIAL, some declarations not delivered within tokenBudget (see each excerpt's
  line range): …`, names such files.
- **`WITHHELD` now means nothing of the file came back.** It reads
  `// WITHHELD, ranked but not delivered within tokenBudget: …` and lists every ranked file not delivered,
  including those ranked after the point where the budget filled.
- **The variable lines name only as many paths as fit**, then `(+N more)` or a bare count, and as a last
  resort fold into one line: `// NOTE: 1 WITHHELD; raise tokenBudget to see which.`
- **The whole response, separators included, now fits the budget.** A call that filled it is packed again
  with room for those lines and carries up to 116 fewer tokens of code; it used to come back 120–150 tokens
  over.
- **Compression is credited on the source actually delivered**, so a budget-bound call reads lower than
  3.0.0 reported for the same request. Lifetime `token_report` totals are not recomputed, so they mix the two
  ways of counting.

`compose_task_prompt`'s code section inherits the declaration-level delivery but prints no header: the only
sign that a file is incomplete is the line range on each excerpt. Captures of all of this are on the
[tool reference](tool-reference.md#get_context).

!!! info "Why it changed"
    A file larger than the remaining budget used to be withheld whole or truncated from the top, so one big
    file could spend the whole budget on members nobody asked about. Whether delivering declarations recovers
    more of the answer is not yet measured: [Benchmarks](benchmarks.md) still reports 3.0.0 figures and says
    so.

---

## 5. `search_code` refuses `topK` above 100, and says when it leaves files stale

**Who it affects:** callers asking for more than 100 hits; anyone searching right after a large change.
**What you do:** ask for 100 or fewer, or use `get_context` or `summarize_repo` for a broad read. When the
header carries a `// NOTE: N indexed file(s) changed on disk and were not re-indexed …` line, run
`index_repo`.

```text
search_code: topK must be at most 100 (got 101). For more of the repository than that, call get_context with the paths these hits name, or summarize_repo. (Parameter 'topK')
```

A search re-embeds at most 25 changed files before answering, so one search after a branch switch no longer
spends minutes embedding; the `NOTE` line counts what it left behind. Hits from Python and Ruby files now
carry `# path:start-end (Kind Symbol)` locators rather than `//`.

---

## 6. `remember` refuses oversized facts

**Who it affects:** anything that stores whole files, long transcripts or long URLs as facts.
**What you do:** store the decision in at most 4,000 characters, a category of at most 64 (after trimming),
and point `source` (at most 2,048) at the file. Facts already stored are untouched, and a blank `source` is
now stored as no source.

```text
remember: remember: text must be at most 4000 characters (got 4001) - a fact is stored permanently and returned by every recall that matches it. There is no tool that deletes a fact, so an oversized one would be permanent. (Parameter 'text')
```

---

## 7. Python and Ruby locators are `#` comments

**Who it affects:** anything parsing `//` locator lines or truncation markers out of Python or Ruby
excerpts, in `get_context` and in `search_code` hits.
**What you do:** accept `#` for those two languages. The response header itself still starts with `//`.

---

## What has not changed

- **Tool, prompt and resource names, and every argument name.** Two argument ranges narrowed (`topK`,
  `remember`'s three fields) and the `get_context` header gained lines; nothing was renamed or removed.
- **Environment variable names.** None added or removed; two narrowed (`SANKSHEP_ALLOWED_HOSTS` is required
  on a routable bind and accepts `*`; API keys must be 32 characters or more).
- **The `.sankshep/` layout.** No index rebuild; `facts.db` and `stats.db` are untouched. `token_report`'s
  lifetime totals are not reset, so they mix 3.0.0 and 4.0.0 crediting.
- **Exit code 1 for every startup refusal**, one line on stderr.
- **Zero egress by default**, the loopback default bind, .NET 10, MCP SDK 2.2.0 and the four protocol
  revisions, and the published platforms (`win-x64`, `linux-x64`, `linux-arm64`, `osx-arm64`).

## Verifying what you pulled

New in 4.0.0, the image and the chart are signed. [Security → supply chain](security.md#supply-chain) gives
the `cosign verify` command; the Helm chart's package is not yet publicly pullable, so for now only the image
can be checked by a reader.

## If something goes wrong

[Troubleshooting](troubleshooting.md) has an entry for each refusal above, and
[Feedback](feedback.md) is the channel for anything it does not cover.
