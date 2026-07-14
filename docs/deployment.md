# Deployment

For an individual developer, `sankshep serve --repo .` over stdio is all you need ([Install](install.md)).
For a shared or long-running service, Sankshep also serves **Streamable HTTP** from the same binary and
ships container / service / Kubernetes options. Everything below is **opt-in and local-first** — no
telemetry by default, no outbound network at rest.

## HTTP mode

```bash
sankshep --http --repo /srv/repo
```

Binds `127.0.0.1:8080` by default (loopback), exposing the same MCP tools over Streamable HTTP plus
`/health/live`, `/health/ready`, and an embedded KPI dashboard at `/dashboard`. Set
`ASPNETCORE_URLS=http://0.0.0.0:8080` to bind a routable address (then turn on auth — below).

## Docker

A multi-arch, non-root image is published to GHCR. Mount your repo **read-only**; state and the model
live on separate volumes:

```bash
docker run --rm -p 8080:8080 \
  -v "$PWD:/repo:ro" \
  -v sankshep-state:/state \
  -v sankshep-models:/models \
  ghcr.io/nitinpawar28/sankshep:latest
```

## Windows Service / systemd

Run `--http` as a native service:

- **Windows:** `sankshep service install --repo C:\path\to\repo` (elevated) registers a Windows Service
  with auto-start and restart-on-crash; logs go to the Event Log.
- **Linux:** install the provided `systemd` unit (`Type=notify`, journald, restart-on-failure).

## Kubernetes

A Helm chart deploys Sankshep as a single-repo-per-release workload with a **git-sync** sidecar keeping
the repository fresh and read-only, plus model/state PVCs. Native sidecar on K8s ≥ 1.29, with a
classic-sidecar fallback for older clusters.

## Air-gapped / zero-egress

Sankshep runs with network egress blocked. Side-load the embedding model and fail closed:

```bash
export SANKSHEP_MODEL_OFFLINE=1   # never attempt a download; error clearly if the model is absent
export SANKSHEP_MODEL_DIR=/models # side-load model.onnx + vocab.txt under <dir>/bge-small-en-v1.5/
sankshep --http --repo /repo
```

The package can also be built from an offline NuGet feed, and the image prebaked with the model, for a
fully disconnected deployment.

## Authentication (opt-in)

The HTTP surface is unauthenticated on loopback by default. For a LAN or enterprise deployment:

- **API key:** `SANKSHEP_API_KEY=<key>` — every request except the health probes needs
  `Authorization: Bearer <key>`.
- **OAuth 2.1 Resource Server:** `SANKSHEP_OAUTH_AUTHORITY` + `SANKSHEP_OAUTH_AUDIENCE` — Sankshep
  validates tokens from *your* identity provider (audience/issuer/lifetime/signature + scopes) and
  serves Protected Resource Metadata. It is a Resource Server only; it never issues or forwards tokens.
- **DNS-rebinding:** `SANKSHEP_ALLOWED_HOSTS` pins the `Host` header for non-loopback binds.

## Telemetry (opt-in, counts-only)

Metrics export is off by default (zero outbound). Point Sankshep at a **customer-controlled** OpenTelemetry
Collector to aggregate KPIs across a fleet:

```bash
export SANKSHEP_OTLP_ENDPOINT=http://otel-collector:4317
```

It exports **counts and histograms only** — never code content or file paths (the only per-metric repo
identifier is a folder name). Leave the endpoint unset for a fully local, zero-egress deployment.
