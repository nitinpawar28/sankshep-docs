# Changelog

Notable changes by release. The current release is **1.8.0**; **2.0.0** is in preparation.

!!! note "Corrected against the tags"

    Two entries below were previously filed under the wrong release, and the largest change in 1.6.0 was
    missing entirely. Each attribution here has been checked with `git tag --contains` against the commit
    that made it.

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
