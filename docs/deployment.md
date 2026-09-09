# Deployment

For an individual developer, `sankshep serve --repo .` over stdio is all you need ([Install](install.md)).
For a shared or long-running service, Sankshep also serves **Streamable HTTP** from the same binary and
ships container / service / Kubernetes options. Everything below is **opt-in and local-first** — no
telemetry by default, no outbound network at rest.

!!! warning "How much of this is verified, and how much is only shipped"

    These tiers do not carry equal evidence, and the difference is worth knowing before you build on one.

    - **stdio** is the supported product. It is what the test suite, the protocol-conformance checks and
      the benchmarks all exercise.
    - **HTTP and the container** are verified in CI: the image is built, run with a read-only repository
      mount, and driven over Streamable HTTP on every change — on **both** amd64 and arm64.
    - **Kubernetes / Helm, the Windows Service, systemd and OAuth** are shipped and tested at the unit and
      template level, but have **not been verified end to end on a real cluster, an elevated Windows
      session, or against a live identity provider.** They are expected to work; nobody has watched them
      work.

    This caveat is required by ADR-0019 and stays until that verification happens.

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
  --read-only --cap-drop=ALL --tmpfs /tmp \
  -v "$PWD:/repo:ro" \
  -v sankshep-state:/state \
  -v sankshep-models:/models \
  -e SANKSHEP_API_KEYS=change-me \
  ghcr.io/nitinpawar28/sankshep:latest
```

`--read-only --cap-drop=ALL` are the flags that actually harden the container, and **an image cannot set
them for you** — they are runtime settings. The process writes only `/models`, `/state` and `/tmp`, which
is why the read-only root filesystem costs nothing here. The Helm chart applies the equivalent
(`readOnlyRootFilesystem`, `drop: [ALL]`) on your behalf.

Clients then send `Authorization: Bearer change-me`. For a trusted, network-isolated host only, swap the
key for `-e SANKSHEP_ALLOW_UNAUTHENTICATED=1` to accept the exposure instead.

## Windows Service / systemd

Run `--http` as a native service:

- **Windows:** `sankshep service install --repo C:\path\to\repo` (elevated) registers a Windows Service
  with auto-start and restart-on-crash; logs go to the Event Log. `sankshep service uninstall` (elevated)
  removes it.
- **Linux:** a `systemd` unit (`Type=notify`, journald, restart-on-failure) — reproduced in full below,
  because the source repository is private and "the provided unit" was not something a reader could
  obtain.

### The systemd unit

Save as `/etc/systemd/system/sankshep.service`, then `systemctl daemon-reload && systemctl enable --now
sankshep`:

```ini
[Unit]
Description=Sankshep MCP server
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/local/bin/sankshep --http --repo /srv/repo
Environment=SANKSHEP_STATE_DIR=/var/lib/sankshep
Environment=SANKSHEP_MODEL_DIR=/var/lib/sankshep/models
Environment=SANKSHEP_API_KEYS=change-me
User=sankshep
Group=sankshep

# Restart, bounded. An unbounded restart loop turns one unusable index into a crash-loop that fills the
# journal while the unit reports "activating".
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=300
StartLimitBurst=3

# Sandboxing. The process needs to read the repository and write its state directory, and nothing else.
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/sankshep
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
LockPersonality=true

[Install]
WantedBy=multi-user.target
```

## Kubernetes

!!! note "The chart is not published yet"

    A Helm chart exists and is tested (`helm lint` plus rendered-template assertions on every change), but
    it lives in the private source repository and **is not published to any registry**, so there is no
    `helm install` command a reader can run. Ask for it, or use the container directly with the
    `docker run` recipe above — the chart's value is the git-sync sidecar and the PVC wiring, not anything
    the image cannot do.

    Publishing it as an OCI chart to GHCR is planned; until then, treating this section as documentation
    of an artifact you can obtain would be wrong.

The chart deploys Sankshep as a single-repo-per-release workload with a **git-sync** sidecar keeping the
repository fresh and read-only, plus model/state PVCs. Native sidecar on K8s ≥ 1.29, with a
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

For a fully disconnected deployment, the two things you can actually do — building from source is not one
of them, because the source is not distributed:

1. **Mirror the packages into your own feed.** Pull `sankshep` and the RID-specific package for your
   platform from nuget.org into your internal NuGet feed, then
   `dotnet tool install -g sankshep --source <your-feed>`.
2. **Mirror the image into your own registry** and pre-bake the model into a derived image. The Helm
   chart's `image.repository` exists to be repointed at it.

On the Helm chart, set `model.offline: true` (which sets `SANKSHEP_MODEL_OFFLINE=1`) alongside a
pre-populated model PVC, or bake the model in and skip the PVC entirely.

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
