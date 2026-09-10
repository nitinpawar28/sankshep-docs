# Environment variables

The single canonical reference for every variable Sankshep reads. Unless noted, a variable is **unset by
default** and the described behavior is off. HTTP/auth variables apply only to `sankshep --http`; stdio mode
ignores them.

!!! info "There is no configuration file"
    Neither transport reads `appsettings.json`. Both take their content root from the installed binary's
    directory and drop every JSON configuration source, so **this page is the whole surface**. The served
    repository is input, never configuration: a file committed by anyone who can write to that repository
    must not be able to change how the server runs.

    In the repository itself the same table lives at `docs/deploy/environment.md`, and a test fails the
    build if `src/` reads a `SANKSHEP_*` name that is not registered, or if the table names one that no
    longer exists.

## Core & state

| Variable | Purpose | Default |
|---|---|---|
| `SANKSHEP_STATE_DIR` | Where per-repo state (`facts.db`, `index.db`, `stats.db`) is written. | `<repo>/.sankshep/` |
| `SANKSHEP_MODEL_DIR` | Directory to cache — or **side-load** — the embedding model (`model.onnx` + `vocab.txt` under `<dir>/bge-small-en-v1.5/`). | per-user cache location |
| `SANKSHEP_MODEL_OFFLINE` | `1` disables the model download entirely — the server **fails closed** if the model isn't already present. Pair with `SANKSHEP_MODEL_DIR` for air-gapped use. | off (download on first use) |
| `SANKSHEP_WATCH` | `1` enables a filesystem watcher that debounces changes and refreshes the index in the background. Off, freshness is maintained verify-on-read. | off |
| `SANKSHEP_DISABLE_VEC0` | `1` forces the pure-C# brute-force vector store instead of the native sqlite-vec `vec0` extension (test seam / ops escape hatch). | off (native `vec0` when available) |
| `SANKSHEP_WARM_MODEL` | **Opt-out, unlike everything else here.** `0` drops the embedding model from the readiness gate, so `/health/ready` returns 200 before the model is loaded. Only the exact string `0` disables it. | **on** |
| `SANKSHEP_MAX_FILE_BYTES` | Source-file size cap. `0` disables the cap. A value that cannot be parsed falls back to the default, never to "no limit". | 2 MiB |

## HTTP transport & authentication (`--http`)

| Variable | Purpose | Default |
|---|---|---|
| `ASPNETCORE_URLS` | Bind address(es). A **non-loopback** value (e.g. `http://0.0.0.0:8080`) triggers the fail-closed auth gate below. | `http://127.0.0.1:8080` (loopback) |
| `SANKSHEP_API_KEYS` | Comma-separated bearer keys (API-key auth). Clients send `Authorization: Bearer <key>`; keys are compared in constant time. `SANKSHEP_API_KEY` is a single-key alias. | unset (auth mode `None`) |
| `SANKSHEP_OAUTH_AUTHORITY` · `SANKSHEP_OAUTH_AUDIENCE` | OAuth 2.1 resource-server mode (validate bearer tokens from your IdP). **Both** are required to select it; setting one alone does nothing. OAuth wins over API-key mode when both are configured. | unset |
| `SANKSHEP_OAUTH_SCOPES` | Scopes a caller's token must carry. **Defaults to `mcp:tools`** — leaving it unset is *not* the same as requiring nothing: a token without that scope is still rejected. | `mcp:tools` |
| `SANKSHEP_OAUTH_RESOURCE` | The protected-resource URI. **Required unless `_AUDIENCE` is an absolute URI** — otherwise the server throws at startup rather than serving unprotected. | derived from `_AUDIENCE` |
| `SANKSHEP_ALLOW_UNAUTHENTICATED` | `1` explicitly permits an **unauthenticated non-loopback bind**. Without it — and without auth configured — a non-loopback bind **refuses to start**. Use only on a trusted, network-isolated deployment. | off (server fails closed) |
| `SANKSHEP_ALLOWED_HOSTS` | Comma-separated `Host`-header allow-list, enforced in the pipeline so health probes stay reachable whatever it says. Entries are compared whole and a port is ignored. | loopback names on a loopback bind; **empty on any other bind**, which accepts no `Host` |
| `SANKSHEP_ALLOWED_ORIGINS` | Comma-separated `Origin` allow-list for browser callers. Empty accepts **no** cross-origin request; a request with no `Origin` at all is unaffected. | empty |

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

## Logging

Everything Sankshep logs goes to **stderr**, on both transports. Under stdio that is required — stdout
carries the JSON-RPC frames — and the HTTP host uses the same stream so the rule is a property of the
process, not of one transport. Docker, Kubernetes, systemd and journald all capture both streams.

These are ASP.NET Core's own variables rather than Sankshep's, so they are not in the tables above, but
they are the ones operators reach for.

| Variable | What it does |
|---|---|
| `Logging__LogLevel__Default` | Sankshep's own log level. `Trace` is safe — see the cap below. |
| `Logging__LogLevel__<category>` | One category, e.g. `Logging__LogLevel__Sankshep=Debug`. |
| `Logging__<provider>__LogLevel__<category>` | Scoped to one provider, e.g. `Logging__Console__LogLevel__Default`. |
| `Logging__Console__FormatterName` | `simple` (default, human-readable) or `json`, one object per line for a log shipper. |

Diagnostics are categorised by the type that emits them, so one noisy source can be quietened without
silencing the rest:

| Category | What it says |
|---|---|
| `Sankshep.Minimizer.ContextMinimizer` | Paths skipped: too large, or linking outside the served root. |
| `Sankshep.Memory.SqliteVectorIndex` | Files skipped during indexing, and why. |
| `Sankshep.Memory.SavingsStatsStore` | `token_report` statistics degraded or reset. **Warning.** |
| `Sankshep.Server.Hosting.ReadinessWarmupService` | Why readiness is holding at 503. **Error.** |
| `Microsoft.AspNetCore` | The framework's per-request lines. **Held at Warning by default.** |
| `ModelContextProtocol` | The MCP SDK. **Capped at Information, and not raisable.** |

```bash
# Structured output for a log shipper.
Logging__Console__FormatterName=json

# Everything Sankshep says about what it skipped and why.
Logging__LogLevel__Sankshep=Debug

# Request logging back on while debugging routing or an ingress.
Logging__LogLevel__Microsoft.AspNetCore=Information
```

!!! danger "Raising the log level cannot make the server print your code — by design"
    The MCP SDK logs **whole JSON-RPC payloads** at `Trace` and `Debug`: every tool argument and result,
    which for this product means source code, search queries and remembered facts. Before 2.0.0,
    `Logging__LogLevel__Default=Trace` — the first thing anyone tries when an MCP server misbehaves —
    echoed all of it to stderr under stdio, and to the log aggregator under HTTP.

    Every `ModelContextProtocol.*` category is now held at `Information` in both hosts, **against every
    spelling**. That matters because the rules compose: a rule naming a provider beats one that does not,
    and a longer category beats a shorter one, so `Logging__Console__LogLevel__Default=Trace` and
    `Logging__LogLevel__ModelContextProtocol.Protocol=Trace` would each re-open it if handled naively.
    Both are handled specifically.

    **There is deliberately no variable that lifts the cap.** Sankshep's own `Trace` logging is
    unaffected, which is the point of capping the SDK rather than raising the floor for the whole process.

**ASP.NET Core request logging is quiet by default.** The framework emits about five `Information` lines
per request, and liveness and readiness probes run every ten to fifteen seconds for the life of a pod, so
an idle server's entire log was health checks. `Microsoft.AspNetCore` is held at `Warning` unless you set a
level for that category yourself; a failing request still speaks up. This is a default, not a cap — request
lines carry a method, a path and a status code, never a payload.

**Under a service manager.** The Windows Service host routes logs to the Event Log, whose provider starts
at `Warning`, so degradations are logged at `Warning` or above to be sure they arrive. Under systemd,
stderr is journald: `journalctl -u sankshep`.

## TLS

| Variable | What it does |
|---|---|
| `Kestrel__Endpoints__Https__Url` | Serve TLS directly instead of behind a terminating proxy. |
| `Kestrel__Certificates__Default__Path` · `__Password` | The certificate to serve it with. |

`ASPNETCORE_URLS` is only **one** of the routes to a bind — `Kestrel__Endpoints__<name>__Url` reaches
Kestrel too and wins over it — which is why the startup guard reads the resolved endpoint set rather than
that variable. Sankshep warns at startup when a credential mode is configured on a routable **plain HTTP**
address, because the bearer token would then cross the network in the clear. It is a warning, not a
refusal: terminating TLS in front of Sankshep is a correct deployment it cannot see from the inside.

## Development / eval harness
<<<NEW
## Development / eval harness

These are read by the benchmark eval harness (`Sankshep.Evals`), **not** the MCP server: `SANKSHEP_SERVER_EXE`
(path to the server binary under test) and the judge credentials `ANTHROPIC_API_KEY` / `OPENAI_API_KEY`
(plus optional `*_BASE_URL` / `*_MODEL`). See [Benchmarks](benchmarks.md).

`SANKSHEP_REQUIRE_MODEL` and `SANKSHEP_REQUIRE_VEC0` are **test-suite** knobs: they make a test run fail
instead of skip when the embedding model or the sqlite-vec native is absent. Nothing in the product reads
them.
