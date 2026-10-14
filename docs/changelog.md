# Changelog

Notable changes by release. The current release is **4.0.0**.

!!! note "Corrected against the tags"

    Two entries below were previously filed under the wrong release, and the largest change in 1.6.0 was
    missing entirely. Each attribution here has been checked with `git tag --contains` against the commit
    that made it.

## 4.0.0

*2026-10-13 — five breaking changes; on the default stdio path most installs feel none of them.*
See [Upgrading to 4.0.0](upgrading-4.0.md). No index rebuilds in this release.

### Breaking changes

- **A routable `--http` bind refuses to start without `SANKSHEP_ALLOWED_HOSTS`.** The server used to accept
  any `Host` when the variable was unset, which left the DNS-rebinding defence off exactly where it matters.
  Set it to the names clients use, or to `*` to accept any `Host` deliberately. A value naming no host, or
  `*` mixed with names, is refused too. The Helm chart sets the list, and renders `*` when
  `allowedHosts.enabled` is `false`.
- **An API key under 32 characters, or only one of `SANKSHEP_OAUTH_AUTHORITY` / `SANKSHEP_OAUTH_AUDIENCE`,
  refuses to start** with one line on stderr and exit 1. One OAuth variable alone used to fall back silently
  to API-key mode or to no authentication.
- **`get_context` keeps its `tokenBudget` ceiling on calls that fill it.** The `WARNING`, `NOTE` and
  `WITHHELD` header lines used to ride on top of the budget, so a budget-bound call came back 120–150 tokens
  over. They now name only as many paths as fit and count the rest, folding into one `// NOTE: … raise
  tokenBudget to see which.` line as a last resort, and such a call is packed again with room for them, so it
  carries up to 116 fewer tokens of code. A call that fits is unchanged. A client that parsed names out of
  those lines should read the count instead.
- **`remember` refuses `text` over 4,000 characters, `category` over 64 and `source` over 2,048.** A fact
  cannot be deleted, so an oversized one inflated every later `recall`, `DECISIONS.md` and composed prompt
  for good. Store the decision, and point `source` at the file.
- **`search_code` refuses `topK` above 100.** Only the lower bound was checked, and a very large value came
  back as an error blaming a locked index. Ask for 100 or fewer; for broader reads use `get_context` or
  `summarize_repo`.

### Changed

- **`get_context` delivers a file that does not fit by its declarations instead of not at all.** When a file
  does not fit what is left of `tokenBudget`, it is cut at its types and methods, the ones the query is about
  first, across every ranked file. Each excerpt is cited `// path:start-end Symbol`. A new `// PARTIAL, …`
  header line names the files that came back in part, and `WITHHELD` now means nothing of the file came back.
  A file that fits whole comes back exactly as before. `compose_task_prompt` uses the same engine.
- **`search_code` re-embeds at most 25 changed files before answering.** After a branch switch one search
  could spend minutes re-embedding. When files are left behind, the header says how many, and `index_repo`
  brings them current.

### Security

- **A link anywhere along a path now counts for containment, not only one at its end.** `get_context`,
  `index_repo` and `summarize_repo` resolve every path to where it really lands, link by link, before reading
  it, and such paths are reported as escaping the served `--repo` root. The index drops a stored key that a
  later checkout has put behind an outbound link instead of re-reading it.
- **The release image and the Helm chart are signed** keylessly with cosign by the release workflow's own
  identity, and the release verifies both signatures before it finishes. See
  [Security → supply chain](security.md#supply-chain) for the `cosign verify` command.
- **A release refuses an image with a fixable HIGH or CRITICAL vulnerability**, and requires green smoke
  results for the tagged commit.
- **A served repository can no longer run a program when the server starts on it.** `git` is now found only
  in `PATH`'s absolute entries, never in the repository or the current directory.
- **`sankshep service install` starts `sc.exe` from the system directory** rather than by bare name.

### Fixed

- **A truncated excerpt is credited with the source it covers, not its whole file**, so compression on a
  budget-bound call reads lower than before and is honest.
- **Locator lines and truncation markers in Python and Ruby excerpts are `#` comments.** They were always
  `//`, which in those languages reads as code.
- **`get_context` names every ranked file it did not deliver**, as "ranked but not delivered"; files ranked
  after the point where the budget filled used to be missing from the list and the count.
- **The budget counts the separators between chunks**, so a budget filled exactly no longer comes back a
  few tokens over.
- **Stripping a comment no longer fuses the tokens around it**, including adjacent comments.
- **`remember` stores a blank `source` as no source.**
- **`search_code` counts only files it really re-indexed** against its 25-file cap.
- **Semantic ranking no longer puts a file the index found below one it never found.**
- **An `index.db` that cannot be opened no longer stops the server**: `index_repo` and `search_code` return an
  error naming the file and what to check, and the other tools work.
- **`summarize_repo` skips a file it cannot read and names it** on a `WARNING:` line instead of failing the
  whole call, and says a file path is a file instead of "path not found".
- **`recall` requires every word of the query**; words past the sixteenth distinct one were ignored.
- **Removing an unused `using` alias at `Aggressive` keeps whatever shares its line.**

### Deployment

- **The Helm chart's readiness probe allows 5 seconds**, like liveness.
- **The observability compose example binds to loopback and Grafana is no longer anonymous**; its `.env` must
  set `GRAFANA_ADMIN_PASSWORD`.
- **The Linux tarballs ship `LICENSE`, `NOTICE` and `THIRD-PARTY-NOTICES.md`**, as the package and image did.

## 3.0.0

*2026-09-20 — three breaking changes, and most installs feel only the first.*
See [Upgrading to 3.0.0](upgrading-3.0.md).

### Removed

- **`osx-x64` is no longer a published platform.** Microsoft.ML.OnnxRuntime ships no native for it, so
  the package installed and then failed the first time anything needed an embedding — `index_repo` and
  `search_code` over stdio, and a server stuck at 503 over HTTP. An Intel Mac on 2.0.0 keeps working and
  cannot update; the container image runs fully on one.

### Changed

- **`compose_task_prompt`'s `tokenBudget` now bounds the code**, with remembered conventions additive on
  their own 600-token budget. It used to be split ~70/30, which ran the underlying engine at 70% of the
  caller's number and cost the composer 0.15 of key-point recall against that engine. The same request
  now returns more code, and the rendered prompt is larger than the number you passed.
- **Every sqlite-vec index rebuilds once**, automatically, on first use. Derived data only: remembered
  facts and savings statistics are untouched.
- **Ranking weights a query term by how rare it is among the files being ranked.** Measured: mean
  compression rose from 56.2% to 59.5% at Balanced and 32.3% to 38.5% at Conservative. **Mean recall did
  not move outside its own run-to-run noise, so no recall gain is claimed** — see the error bar on
  [Benchmarks](benchmarks.md).

### Security

- **The `vec0` native is upstream's own release, verified by SHA-256 and loaded by absolute path.** The
  package previously used carried binaries matching no upstream release asset of any version.
- **Every build restores exactly the dependency graph CI tested.** Each project commits a lock file, and
  CI, the release and the image build all restore in locked mode.

## 2.0.0 — 2026-09-11 · Audit remediation

A production-readiness audit of 1.8.0 found **189 issues across thirteen dimensions and scored it 45/100 —
not ready**, including three critical defects that produced silently wrong output on the default path. This
release is the remediation of that audit: 162 findings closed, 23 refuted on re-examination, and four
deferred with their reasons stated.

**Start here if you are upgrading:** [Upgrading to 4.0.0](upgrading-4.0.md) lists what the current release asks of you. Coming from 2.0.0, read [Upgrading to 3.0.0](upgrading-3.0.md) as well; coming from 1.x, start at [Upgrading to 2.0.0](upgrading-2.0.md) — the 1.x and 2.0.0 index rebuilds both apply, and 4.0.0 adds none.

**The three criticals — all of them produced output that looked correct:**

- **A leading same-line comment deleted the code after it.** `/* note */ DoWork();` lost `DoWork()`, in
  every language and at every minimization level. Reproduced in C#, TypeScript, Go and Java.
- **A Rust doc comment deleted the signature beneath it.** A `///` line took the `fn` or `struct` with it.
- **An index built on a subdirectory could never be read back.** Chunks were keyed relative to that
  subdirectory, so the next `search_code` resolved them against the repo root and deleted every row — and
  `index_repo --force` rewrote exactly the same unusable keys.

**Results you may have to adapt to:**

- **`search_code` returns located code, not JSON.** Hits used to arrive as one escaped JSON line; they now
  use the same shape `get_context` does. **Breaking for a scripted caller.**
- **An empty index is an error.** It used to be `{count:0,results:[]}`, indistinguishable from "this code
  does not exist".
- **`summarize_repo` covers every language Sankshep parses, not just C#,** and takes a `maxTokens` ceiling
  (default 20,000) that it reports when it truncates. It previously had no budget at all.
- **`recall` caps its result set at 50** and reports `matched` and `truncated` so you can tell "all of
  them" from "the first fifty".
- **`index_repo` reports what the walk did**, not the whole index's size, and emits progress per file.
- **Failures arrive as `isError` with an actionable message** rather than a generic string or a clean
  empty success.

**Security and privacy:**

- **`Host` and `Origin` are validated on every non-health request.** The documented "loopback blocks DNS
  rebinding" claim was previously false.
- **Raising the log level can no longer print your code.** The MCP SDK logs whole JSON-RPC payloads at
  `Trace`; every `ModelContextProtocol.*` category is now capped at `Information` against every spelling,
  with no variable to lift it.
- **`appsettings.json` inside a served repository is no longer configuration,** on either transport. It
  could previously re-bind the server to `0.0.0.0`.
- **An absolute path is refused on HTTP,** and by `index_repo` on both transports.
- **`service install` refuses a user-writable binary** — which is where `dotnet tool install -g` puts one.
- **A fail-closed refusal exits 1** with one line on stderr, instead of `0xE0434352` and a stack trace.

**Chart 1.3.0** renders the `Host` allow-list by default, drops an unused API token from the pod, and gives
the liveness probe a workable timeout. **Breaking for Ingress users** — see the upgrade page.

Runtime moves to **.NET 10** and the MCP SDK to **2.2.0**; the server answers protocol revisions
`2024-11-05`, `2025-03-26`, `2025-06-18` and `2025-11-25`.

## 1.8.0 — 2026-07-18 · Security hardening

An adversarial security audit (ten dimensions) found **no critical or high-severity** issues. This release
remediates the medium/low findings it surfaced:

- **Fail-closed HTTP auth.** A non-loopback bind (`ASPNETCORE_URLS=0.0.0.0`) with auth mode `None` now
  **refuses to start** unless authentication is configured or `SANKSHEP_ALLOW_UNAUTHENTICATED=1` is set. The
  Helm chart gains a secure-by-default `auth` block.
- **Untrusted-repo hardening.** ReDoS-safe `.gitignore` matching; relative `../..` paths confined to the
  served `--repo`; `.docx`/`.pdf` extraction size-capped against decompression bombs; directory walks are
  iterative and symlink-cycle-safe.
- **Private disclosure channel.** Added `SECURITY.md` with GitHub private vulnerability reporting.

See [Security & privacy](security.md).

## 1.7.0 — 2026-07-18 · Query-targeted minimization

- **Balanced now keeps the code that answers your query.** At the default `Balanced` level, `get_context`
  keeps the method bodies relevant to your (stemmed) query and collapses the rest — the targeted middle
  between `Conservative` (keep all) and `Aggressive` (collapse all).
- **Published, measured benchmarks** on a real codebase. See [Benchmarks](benchmarks.md).

## 1.6.0 — 2026-07-17 · Seven more languages, and the rest of path anchoring

- **Parse-aware support for seven more languages** — Go, Java, C, C++, Rust, PHP and Ruby, plus hardened
  TypeScript. This was the largest change in the release and was missing from this page entirely.
- **`summarize_repo` and `index_repo` resolve relative paths against the served `--repo` root.**
  `get_context` had already been anchored in 1.2.0; this completed it.
- **Composer conventions are drawn by category**, and memory records UTC timestamps and echoes `"global"`
  for an unscoped branch.

## 1.4.0 — 2026-07-16 · Result shapes

- **Breaking:** tool results changed shape. Callers parsing the previous form need updating. This was not
  recorded at the time and is noted here rather than left out.

## 1.3.0 — 2026-07-16 · Honest accounting

- **Honest savings accounting** — compression is measured against the size of the files actually
  delivered; the amount *searched* is disclosed but never counted as a saving. *(Previously listed under
  1.6.0. The commit is in v1.3.0.)*

## 1.2.0 — 2026-07-15 · `get_context` path anchoring

- **`get_context` resolves relative `paths` against the served `--repo` root**, not the client's working
  directory — previously a relative path could silently target the wrong tree. *(Previously listed under
  1.6.0. The commit is in v1.2.0.)*

## 1.1.0 — 2026-07-15 · Result shapes

- **Breaking:** an earlier result-shape change, likewise unrecorded at the time.

## 1.0.0 – 1.0.2 — 2026-07-15 · Foundations

Initial public releases: the ten-primitive MCP surface (eight tools, the `compose_task_prompt` prompt and
one resource), local-first embeddings (ONNX Runtime) and vector search (sqlite-vec), per-repo
branch-scoped memory, and the HTTP transport / deployment tier.

!!! info "There is no 1.5.0"

    The tags go 1.4.0 → 1.6.0. Nothing was released as 1.5.0; the gap is not a missing entry.
