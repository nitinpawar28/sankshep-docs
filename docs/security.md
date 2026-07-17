# Security & privacy

Sankshep is built to run entirely on your machine. This page states exactly what it stores, where, and what —
if anything — leaves the host.

## Data handling: what leaves your machine

**By default, nothing.** No code, file paths, queries, or memory content is sent anywhere. Embeddings are
computed locally via ONNX Runtime; vector search is local via sqlite-vec.

The **only** outbound network activity is:

| Activity | When | What is sent |
|---|---|---|
| Embedding-model download | First `index_repo`/`search_code`, unless side-loaded | A model-file request to the model host — no repo content. Suppress with `SANKSHEP_MODEL_OFFLINE=1` + `SANKSHEP_MODEL_DIR`. |
| Telemetry export | **Only** if you set `SANKSHEP_OTLP_ENDPOINT` or `SANKSHEP_PROMETHEUS` | Counts and a low-cardinality repo tag (the folder name) — **never** file paths, queries, or code. To a collector **you** run. |

There is **no telemetry by default** and no dollar/billing phone-home — Sankshep is never told which model you
use or what you pay (ADR-0011, ADR-0017).

## Where state lives on disk

Per-repo state is written under `<repo>/.sankshep/` (override the location with `SANKSHEP_STATE_DIR`):

| File | Written by | Contents |
|---|---|---|
| `facts.db` | `remember` | Your remembered facts + their git-branch tags and optional source. |
| `index.db` | `index_repo` | Code chunks and their local embedding vectors. |
| `stats.db` | `get_context`, `compose_task_prompt` | Per-tool token counts (no code, no paths). |

The embedding model is cached under `SANKSHEP_MODEL_DIR` (or a default per-user location). None of this is
transmitted; deleting `.sankshep/` resets a repo's index, memory, and stats.

!!! note "Add `.sankshep/` to your `.gitignore`"
    It's local state, not source. Keep `facts.db`/`index.db`/`stats.db` out of version control.

## Network exposure (HTTP mode)

`sankshep --http` binds `http://127.0.0.1:8080` — **loopback only** — by default, a DNS-rebinding defense.
Accepting non-loopback traffic requires explicitly setting `SANKSHEP_ALLOWED_HOSTS` **and** `ASPNETCORE_URLS`.
In HTTP mode you can require auth (`SANKSHEP_API_KEY`/`SANKSHEP_API_KEYS` for bearer keys, or the OAuth 2.1
resource-server mode via `SANKSHEP_OAUTH_*`); health probes stay anonymous. stdio mode has no network surface
at all.

## Supply chain

Container images are published to GHCR. Verifying image provenance (signature/SBOM) is recommended before
running in production; if a signed-image/SBOM workflow is not yet published for your version, pin by digest and
build from the released binary you trust.

## Reporting a vulnerability

Please report suspected security issues **privately** — do **not** open a public issue, which would disclose
the problem before a fix exists. Use the maintainer contact on the [Feedback](feedback.md) page and mark the
report as security-sensitive.

> **Maintainer:** set a dedicated private security contact (e.g. a `security@` address or a GitHub private
> vulnerability-reporting channel) and replace this line, so reporters have an unambiguous private path.
