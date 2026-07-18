# Environment variables

The single canonical reference for every variable Sankshep reads. Unless noted, a variable is **unset by
default** and the described behavior is off. HTTP/auth variables apply only to `sankshep --http`; stdio mode
ignores them.

## Core & state

| Variable | Purpose | Default |
|---|---|---|
| `SANKSHEP_STATE_DIR` | Where per-repo state (`facts.db`, `index.db`, `stats.db`) is written. | `<repo>/.sankshep/` |
| `SANKSHEP_MODEL_DIR` | Directory to cache — or **side-load** — the embedding model (`model.onnx` + `vocab.txt` under `<dir>/bge-small-en-v1.5/`). | per-user cache location |
| `SANKSHEP_MODEL_OFFLINE` | `1` disables the model download entirely — the server **fails closed** if the model isn't already present. Pair with `SANKSHEP_MODEL_DIR` for air-gapped use. | off (download on first use) |
| `SANKSHEP_WATCH` | `1` enables a filesystem watcher that debounces changes and refreshes the index in the background. Off, freshness is maintained verify-on-read. | off |
| `SANKSHEP_DISABLE_VEC0` | `1` forces the pure-C# brute-force vector store instead of the native sqlite-vec `vec0` extension (test seam / ops escape hatch). | off (native `vec0` when available) |

## HTTP transport & authentication (`--http`)

| Variable | Purpose | Default |
|---|---|---|
| `ASPNETCORE_URLS` | Bind address(es). A **non-loopback** value (e.g. `http://0.0.0.0:8080`) triggers the fail-closed auth gate below. | `http://127.0.0.1:8080` (loopback) |
| `SANKSHEP_API_KEYS` | Comma-separated bearer keys (API-key auth). Clients send `Authorization: Bearer <key>`; keys are compared in constant time. `SANKSHEP_API_KEY` is a single-key alias. | unset (auth mode `None`) |
| `SANKSHEP_OAUTH_AUTHORITY` · `_AUDIENCE` · `_RESOURCE` · `_SCOPES` | OAuth 2.1 resource-server mode (validate bearer tokens from your IdP). | unset |
| `SANKSHEP_ALLOW_UNAUTHENTICATED` | `1` explicitly permits an **unauthenticated non-loopback bind**. Without it — and without auth configured — a non-loopback bind **refuses to start**. Use only on a trusted, network-isolated deployment. | off (server fails closed) |
| `SANKSHEP_ALLOWED_HOSTS` | Comma-separated `Host`-header allow-list (DNS-rebinding hardening) for non-loopback binds. | unset (Host unrestricted; the loopback default is the protection) |

!!! warning "Fail-closed bind"
    On a non-loopback bind with auth mode `None`, the server **refuses to start** unless
    `SANKSHEP_ALLOW_UNAUTHENTICATED=1`. Configure `SANKSHEP_API_KEYS` (or `SANKSHEP_OAUTH_*`) instead
    whenever you can. `SANKSHEP_ALLOWED_HOSTS` is additional hardening — it does **not** satisfy the auth gate.

## Telemetry (opt-in)

| Variable | Purpose | Default |
|---|---|---|
| `SANKSHEP_OTLP_ENDPOINT` | OTLP endpoint to **push** metrics to (a collector **you** run). Counts + a low-cardinality repo-folder tag only — never code, paths, or queries. | unset (no outbound) |
| `SANKSHEP_PROMETHEUS` | `1` serves a `/metrics` **scrape** endpoint (inbound/pull, for your Prometheus). | off |
| `SANKSHEP_FLEET_TEAM` · `SANKSHEP_FLEET_INSTANCE` | Optional labels (`sankshep.team`, `service.instance.id`) added to exported metrics. | unset |

## Development / eval harness

These are read by the benchmark eval harness (`Sankshep.Evals`), **not** the MCP server: `SANKSHEP_SERVER_EXE`
(path to the server binary under test) and the judge credentials `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`
(plus optional `*_BASE_URL` / `*_MODEL`). See [Benchmarks](benchmarks.md).
