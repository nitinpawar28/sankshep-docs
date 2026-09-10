# Changelog

Notable changes by release. The current release is **2.0.0**.

!!! note "Corrected against the tags"

    Two entries below were previously filed under the wrong release, and the largest change in 1.6.0 was
    missing entirely. Each attribution here has been checked with `git tag --contains` against the commit
    that made it.

## 2.0.0 — 2026-09-10 · Audit remediation

A production-readiness audit of 1.8.0 found **189 issues across thirteen dimensions and scored it 45/100 —
not ready**, including three critical defects that produced silently wrong output on the default path. This
release is the remediation of that audit: 162 findings closed, 23 refuted on re-examination, and four
deferred with their reasons stated.

**Start here if you are upgrading:** [Upgrading to 2.0.0](upgrading-2.0.md) lists the one-time steps.

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

## 1.2.0 — 2026-07-16 · `get_context` path anchoring

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
