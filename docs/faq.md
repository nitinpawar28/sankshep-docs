# FAQ & gotchas

Short answers to the first-run surprises — most are discoverable only by hitting them otherwise.

## The first `index_repo` is slow, or downloads a lot

The first time you use embeddings (`index_repo`, or `search_code` against a non-empty index), Sankshep
downloads the local embedding model (~127 MB, once, checksum-verified). It is fully offline afterwards. For
air-gapped hosts, side-load the model and set `SANKSHEP_MODEL_OFFLINE=1` — see [Deployment](deployment.md#air-gapped-zero-egress).

## My container / Helm pod won't start — "Refusing to start … no authentication"

Since 1.8.0 the HTTP server **fails closed** on a non-loopback bind (`0.0.0.0`) when auth is `None`. Configure
`SANKSHEP_API_KEYS` (or `SANKSHEP_OAUTH_*`), or — only on a trusted, network-isolated deployment — set
`SANKSHEP_ALLOW_UNAUTHENTICATED=1`. See [Security → network exposure](security.md#network-exposure-http-mode).

## I ran `index_repo`, but `search_code` returns nothing

Confirm you indexed the **same** repo the server serves, that your files are a
[supported type](usage.md), then re-run `index_repo`. Deleting `<repo>/.sankshep/index.db` forces a clean
rebuild. See [Troubleshooting](troubleshooting.md).

## `summarize_repo` returns almost nothing for my Go / TS / Python repo

Not since 2.0.0 — it covers **every language on the [languages table](usage.md)**, not just C#. If a summary
still looks thin, check the second line of the response: it says `TRUNCATED: N of M files shown` when the
`maxTokens` budget stopped the walk. Raise `maxTokens`, summarize a subdirectory, or use `search_code` to go
straight to what you need.

## My HTML / SCSS / template files aren't in the results

Sankshep minimizes and indexes the source languages on the [languages table](usage.md), plus `.docx`/`.pdf`
documents. HTML, SCSS, and framework templates are not currently indexed or minimized.

## `get_context` returned nothing for a file outside my repo

That is deliberate, and in 2.0.0 the boundary is tighter than it was. Relative paths are confined to the
served `--repo` root — a `../..` escape is refused, not read — and the refusal now says so instead of
returning an empty result.

An absolute path is **not** a general escape hatch:

| Transport | Absolute path inside the root | Absolute path outside the root |
|---|---|---|
| stdio | accepted | refused |
| HTTP | refused | refused |

`index_repo` refuses an absolute path on **both** transports. If you need a file that lives outside the
repository, serve a root that contains it, or copy it in. See [Security](security.md).

## Which minimization level should I use?

`Balanced` (the default) for almost everything — it keeps the code relevant to your query and collapses the
rest. `Conservative` when you want every body verbatim (e.g. an exhaustive review). `Aggressive` only when you
want structure and signatures, not behavior — it collapses answer bodies too. See [How it works](how-it-works.md).
