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
`ASPNETCORE_URLS=http://0.0.0.0:8080` to bind a routable address — but a non-loopback bind **refuses to
start** unless you configure authentication (`SANKSHEP_API_KEYS` / `SANKSHEP_OAUTH_*`, see below) or set
`SANKSHEP_ALLOW_UNAUTHENTICATED=1` for a trusted, network-isolated host.

## Docker

A multi-arch, non-root image is published to GHCR. Mount your repo **read-only**; state and the model
live on separate volumes. The image binds `0.0.0.0` (required in a container), so it **fails closed** at
startup unless you configure auth or explicitly accept an unauthenticated bind:

```bash
docker run --rm -p 8080:8080 \
  -v "$PWD:/repo:ro" \
  -v sankshep-state:/state \
  -v sankshep-models:/models \
  -e SANKSHEP_API_KEYS=change-me \
  ghcr.io/nitinpawar28/sankshep:latest
```

Clients then send `Authorization: Bearer change-me`. For a trusted, network-isolated host only, swap the
key for `-e SANKSHEP_ALLOW_UNAUTHENTICATED=1` to accept the exposure instead.

## Windows Service / systemd

Run `--http` as a native service:

- **Windows:** `sankshep service install --repo C:\path\to\repo` (elevated) registers a Windows Service
  with auto-start and restart-on-crash; logs go to the Event Log. `sankshep service uninstall` (elevated)
  removes it.
- **Linux:** install the provided `systemd` unit (`Type=notify`, journald, restart-on-failure).

## Kubernetes

A Helm chart deploys Sankshep as a single-repo-per-release workload with a **git-sync** sidecar keeping
the repository fresh and read-only, plus model/state PVCs. Native sidecar on K8s ≥ 1.29, with a
classic-sidecar fallback for older clusters.

The pod binds `0.0.0.0`, so like the container it **fails closed**: set `auth.apiKeys` (recommended —
source them from a `Secret`) or `auth.allowUnauthenticated=true` for a trusted network. A default
`helm install` with both unset will not start.

## Air-gapped / zero-egress

Sankshep runs with network egress blocked. Side-load the embedding model and fail closed:

```bash
export SANKSHEP_MODEL_OFFLINE=1   # never attempt a download; error clearly if the model is absent
export SANKSHEP_MODEL_DIR=/models # side-load model.onnx + vocab.txt under <dir>/bge-small-en-v1.5/
sankshep --http --repo /repo
```

The package can also be built from an offline NuGet feed, and the image prebaked with the model, for a
fully disconnected deployment. On the Helm chart, set `model.offline: true` (which sets
`SANKSHEP_MODEL_OFFLINE=1`) alongside a pre-populated model PVC.

## Authentication

The HTTP surface is unauthenticated on the loopback default. On any **non-loopback bind** it is
**mandatory**: the server refuses to start unless you configure one of the modes below or set
`SANKSHEP_ALLOW_UNAUTHENTICATED=1` to deliberately accept an unauthenticated bind on a trusted,
network-isolated host. For a LAN or enterprise deployment:

- **API key:** `SANKSHEP_API_KEYS=<key>` (comma-separated for multiple keys; the singular
  `SANKSHEP_API_KEY` remains a single-key alias) — every request except the health probes needs
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
