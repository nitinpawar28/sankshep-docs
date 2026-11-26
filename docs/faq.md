# FAQ & gotchas

Short answers to the first-run surprises — most are discoverable only by hitting them otherwise.

## The first `index` is slow, or downloads a lot

The first time you use embeddings (`index`, or a semantic `find` against a non-empty index), Sankshep
downloads the local embedding model (~127 MiB, once, checksum-verified). It is fully offline afterwards. For
air-gapped hosts, side-load the model and set `SANKSHEP_MODEL_OFFLINE=1` — see [Deployment](deployment.md#air-gapped-zero-egress).

## My container / Helm pod won't start — "Refusing to start … no authentication"

Since 1.8.0 the HTTP server **fails closed** on a non-loopback bind (`0.0.0.0`) when auth is `None`. Configure
`SANKSHEP_API_KEYS` (or `SANKSHEP_OAUTH_*`), or — only on a trusted, network-isolated deployment — set
`SANKSHEP_ALLOW_UNAUTHENTICATED=1`. Since 4.0.0 a container also needs `SANKSHEP_ALLOWED_HOSTS` (the Helm
chart sets it), every API key must be at least 32 characters, and OAuth needs both of its variables or
neither — each refusal is one line on stderr, quoted on [Troubleshooting](troubleshooting.md). See also
[Security → network exposure](security.md#network-exposure-http-mode).

## I ran `index`, but a semantic `find` returns nothing

Confirm you indexed the **same** repo the server serves, that your files are a
[supported type](usage.md), then re-run `index`. Deleting `<repo>/.sankshep/index.db` forces a clean
rebuild. `find` with `mode=exact`, `regex` or `symbol` needs no index at all, so it is the quickest way to
check whether the code is there. See [Troubleshooting](troubleshooting.md).

## My old tool names stopped working after upgrading

5.0.0 renamed every tool and keeps none of the old names as an alias: `get_context` is now `read`,
`search_code` is `find`, `summarize_repo` is `outline`, `index_repo` is `index`, `token_report` is
`report`, `export_decisions` is replaced by `recall` with `format=markdown`, and the `compose_task_prompt`
prompt is `implement`. See [Upgrading to 5.0.0](upgrading-5.0.md).

## `outline` returns almost nothing for my Go / TS / Python repo

Not since 2.0.0 — it covers **every language on the [languages table](usage.md)**, not just C#. If an outline
still looks thin, check the second line of the response: when the `maxTokens` ceiling (default 4,000, over
the whole answer) cut the outline, it says so there. Raise `maxTokens`, outline a subdirectory, or use
`find` to go straight to what you need.

## My HTML / SCSS / template files aren't in the results

Sankshep minimizes and indexes the source languages on the [languages table](usage.md); the index also
covers Markdown, manifests and `.docx`/`.pdf` documents. HTML, SCSS and framework templates are not
currently indexed or minimized, but `read` returns any text file you name, those included, as written and
numbered.

## `read` returned nothing for a file outside my repo

That is deliberate, and in 2.0.0 the boundary is tighter than it was. Relative paths are confined to the
served `--repo` root — a `../..` escape is refused, not read — and the refusal now says so instead of
returning an empty result.

An absolute path behaves differently on the two transports:

| Transport | Absolute path inside the root | Absolute path outside the root |
|---|---|---|
| stdio | accepted | accepted: a local client can already read the filesystem |
| HTTP | accepted | refused |

`read`, `outline` and `find`'s `paths` follow this table. `index` refuses any path outside the root on
**both** transports, because the index is keyed to the root. Over HTTP, to work with a file outside the
repository, serve a root that contains it, or copy it in. See [Security](security.md).

## Which minimization level should I use?

`Balanced` (the default) for almost everything — it keeps the code relevant to your query and collapses the
rest. `Conservative` when you want every body verbatim (e.g. an exhaustive review). `Aggressive` only when you
want structure and signatures, not behavior — it collapses answer bodies too. See [How it works](how-it-works.md).
