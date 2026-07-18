# Changelog

Notable changes by release. The current release is **1.8.0**.

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

## 1.6.0 — 2026-07-17 · Path anchoring & honest accounting

- **Path-taking tools resolve relative paths against the served `--repo` root**, not the client's working
  directory — previously a relative path could silently target the wrong tree.
- **Honest savings accounting** — compression is measured against the size of the files actually delivered;
  the amount *searched* is disclosed but never counted as a saving.

## 1.0.0 – 1.4.0 — 2026-07-15/16 · Foundations

Initial public releases: the nine-primitive MCP surface (eight tools plus the `compose_task_prompt` prompt),
local-first embeddings (ONNX Runtime) and vector search (sqlite-vec), per-repo branch-scoped memory, and the
HTTP transport / deployment tier.
