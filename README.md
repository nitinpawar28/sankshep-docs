# sankshep-docs

Public documentation for **Sankshep** — a token-minimized, relevance-ranked codebase-context server
(plus a persistent per-repo memory) for MCP clients.

**📖 Live site: <https://nitinpawar28.github.io/sankshep-docs/>**

The Sankshep product itself is proprietary and free to install
([NuGet](https://www.nuget.org/packages/sankshep) / GHCR). This repository holds only the public
usage documentation and a curated architecture overview; it is built with
[MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and deployed to GitHub Pages.

## Feedback

Bug reports and feature suggestions are welcome — please
[open an issue](https://github.com/nitinpawar28/sankshep-docs/issues) here. (Sankshep's product source
is private and does not take external pull requests.)

## Local preview

```bash
pip install "mkdocs-material>=9.5"
mkdocs serve   # http://127.0.0.1:8000
```

## Maintaining these docs (redaction boundary)

This is a **public** repository. Only usage docs + a *curated* architecture slice belong here. When
updating from the private product repo, the following **never leave the private repo**:

- `docs/plan/**` — internal roadmaps, phase plans, and the risk registers.
- The session playbook and any unshipped/internal ADRs (only a curated subset of architecture is
  summarized in [`docs/architecture.md`](docs/architecture.md), not the raw ADRs).
- Anything implementation-sensitive (internal file layouts, security specifics, unreleased design).

Review the diff before publishing: if a page reveals *how it's built* beyond the curated overview,
trim it.

## License

Documentation content: Copyright © 2026 Nitin Pawar. The Sankshep software is governed by its own
proprietary license (see the product's LICENSE).
