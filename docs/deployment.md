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
    - **Kubernetes / Helm** is installed by CI into a throwaway kind (Kubernetes-in-Docker) cluster on
      every change to the chart or to anything the image is built from: the pod has to become ready and
      answer an authenticated MCP `initialize` through its Service. That run turns git-sync off and loads
      no embedding model, so neither the sidecar syncing a repository nor the model volume holding a model
      has been verified, and the chart has not been verified end to end on a production cluster.
    - **The Windows Service, systemd and OAuth** are shipped and tested at the unit level, but have **not
      been verified end to end in an elevated Windows session, under systemd, or against a live identity
      provider.** They are expected to work; nobody has watched them work.

    This caveat is required by ADR-0019 and stays until that verification happens.

## HTTP mode

```bash
sankshep --http --repo /srv/repo
```

Binds `127.0.0.1:8080` by default (loopback), exposing the same MCP tools over Streamable HTTP plus
`/health/live`, `/health/ready`, and an embedded KPI dashboard at `/dashboard`. Set
`ASPNETCORE_URLS=http://0.0.0.0:8080` to bind a routable address — but a non-loopback bind **refuses to
start** unless you configure authentication (`SANKSHEP_API_KEYS` / `SANKSHEP_OAUTH_*`, see below) or set
`SANKSHEP_ALLOW_UNAUTHENTICATED=1` for a trusted, network-isolated host, **and**, since 4.0.0, set
`SANKSHEP_ALLOWED_HOSTS` to the host names clients use (or `*` to accept any `Host` deliberately). Every API
key must be at least 32 characters.

## Docker

A multi-arch, non-root image is published to GHCR. Mount your repo **read-only**; state and the model
live on separate volumes. The image binds `0.0.0.0` (required in a container), so it **fails closed** at
startup unless you configure auth or explicitly accept an unauthenticated bind — and, since 4.0.0, unless
`SANKSHEP_ALLOWED_HOSTS` names the hosts clients use. The key must be at least 32 characters:

```bash
KEY=$(openssl rand -hex 32)
docker run --rm -p 127.0.0.1:8080:8080 \
  --read-only --cap-drop=ALL --tmpfs /tmp \
  -v "$PWD:/repo:ro" \
  -v sankshep-state:/state \
  -v sankshep-models:/models \
  -e SANKSHEP_API_KEYS="$KEY" \
  -e SANKSHEP_ALLOWED_HOSTS=localhost,127.0.0.1 \
  ghcr.io/nitinpawar28/sankshep:v4.0.0
```

!!! note "Why a version and not `:latest`"
    `:latest` exists and moves with every release, which makes it fine for a throwaway try and wrong for
    anything you want to reproduce — two machines, or the same machine a week apart, can get different
    bytes under that name. Pin the version, or pin a digest
    (`ghcr.io/nitinpawar28/sankshep@sha256:...`) if you need the strongest guarantee. The image carries SLSA
    provenance and an SPDX SBOM, so `docker buildx imagetools inspect` will tell you what you pulled.

`--read-only --cap-drop=ALL` are the flags that actually harden the container, and **an image cannot set
them for you** — they are runtime settings. The process writes only `/models`, `/state` and `/tmp`, which
is why the read-only root filesystem costs nothing here. The Helm chart applies the equivalent
(`readOnlyRootFilesystem`, `drop: [ALL]`) on your behalf.

Clients then send `Authorization: Bearer $KEY`. To reach the container from another machine, add the host
name those clients use to `SANKSHEP_ALLOWED_HOSTS` and publish the port on that interface. For a trusted,
network-isolated host only, swap the key for `-e SANKSHEP_ALLOW_UNAUTHENTICATED=1` to accept the exposure
instead; the allow-list is still required.

## Windows Service / systemd

Run `--http` as a native service:

- **Windows:** `sankshep service install --repo C:\path\to\repo` (elevated) registers a Windows Service
  with auto-start and restart-on-crash; logs go to the Event Log. `sankshep service uninstall` (elevated)
  removes it.

!!! warning "Changed in 2.0.0 — `service install` refuses a user-writable binary"
    A service registered by an administrator runs as `LocalSystem`, so anyone who can replace its
    executable can run code as `LocalSystem`. `dotnet tool install -g` puts the binary under your **user
    profile**, which you can write — so installing a service from there handed that privilege to your own
    account, and to anything running as you.

    The installer now refuses that and names the principal that could replace the file. **Copy the binary
    somewhere only administrators can write and install from there**, e.g. `C:\Program Files\Sankshep\`.

!!! info "Changed in 2.0.0 — a supervised service keeps state outside the repository"
    Under the Windows SCM or systemd, `facts.db`, `index.db` and `stats.db` move from `<repo>/.sankshep`
    to a machine-wide root (`%ProgramData%\Sankshep` on Windows), and the embedding model moves out of the
    service account's profile. Startup names both paths.

    **The index rebuilds and authored memory is not migrated.** Copy `facts.db` across by hand if you want
    to keep it — nothing copies it for you, because copying a file out of an attacker-controllable
    directory while running as `LocalSystem` is the defect the move exists to close.
- **Linux:** a `systemd` unit (`Type=exec`, journald, restart-on-failure, sandboxed) — reproduced in full
  below, exactly as 4.0.0 ships it, because the source repository is private and "the provided unit" was
  not something a reader could obtain.

### The systemd unit

The unit expects the executable at `/opt/sankshep/sankshep` and a dedicated `sankshep` user; it binds
loopback, so it needs no key. Save the unit below as `sankshep.service`, install the tool, create the user and
the state directories, edit `ExecStart`'s `--repo` path, then enable it:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin sankshep
sudo dotnet tool install --tool-path /opt/sankshep sankshep
sudo install -Dm644 sankshep.service /etc/systemd/system/sankshep.service
sudo mkdir -p /var/lib/sankshep/models /var/lib/sankshep/state
sudo chown -R sankshep /var/lib/sankshep
sudo chmod 700 /var/lib/sankshep /var/lib/sankshep/state /var/lib/sankshep/models
sudo systemctl daemon-reload
sudo systemctl enable --now sankshep
curl -fsS http://127.0.0.1:8080/health/ready   # `active` means running, not ready
```

The `dotnet tool` install needs the .NET 10 SDK on the host. (Each release also builds self-contained Linux
tarballs that land the same executable at the same path, but they are attached to a GitHub release in the
private source repository and are **not publicly obtainable**; the nuget.org package is.) To expose the
service beyond loopback, set `ASPNETCORE_URLS`, an authentication mode and `SANKSHEP_ALLOWED_HOSTS` as under
[Authentication](#authentication), and grant any moved state or model path back in `ReadWritePaths`, because
`ProtectSystem=strict` makes everything else read-only.

```ini
[Unit]
Description=Sankshep MCP Server
Wants=network-online.target
After=network-online.target

[Service]
Type=exec
ExecStart=/opt/sankshep/sankshep --http --repo /srv/repo
User=sankshep
UMask=0077
Environment=SANKSHEP_MODEL_DIR=/var/lib/sankshep/models
Environment=SANKSHEP_STATE_DIR=/var/lib/sankshep/state
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=300
StartLimitBurst=3

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
StateDirectory=sankshep
ReadWritePaths=-/var/lib/sankshep
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX

[Install]
WantedBy=multi-user.target
```

The unit above is the one 4.0.0 ships, directive for directive; its comments are left out. Earlier versions of this page
carried five further sandbox directives that were never in the shipped unit - `ProtectKernelTunables`,
`ProtectKernelModules`, `ProtectControlGroups`, `RestrictSUIDSGID` and `LockPersonality`. They are standard systemd
hardening and nothing in Sankshep needs them removed, but, like the unit itself (see the caveat at the top of this page),
they have not been tested under systemd, which is why they are not reproduced here.

## Kubernetes

!!! note "The chart is not publicly available yet"

    A Helm chart exists and is tested (`helm lint` plus rendered-template assertions on every change, and
    the kind install described at the top of this page). Releases from 3.0.0 on push it to GHCR as an OCI
    chart, versioned with the application (the 4.0.0 release pushed chart 4.0.0, signed with cosign), but
    that package **is not publicly pullable yet**, so there is still no `helm install` command a reader can
    run. Ask for it, or use the container directly with the `docker run` recipe above — the chart's value
    is the git-sync sidecar and the PVC wiring, not anything the image cannot do.

    Until the chart package is public, treating this section as documentation of an artifact you can
    obtain would be wrong.

The chart deploys Sankshep as a single-repo-per-release workload with a **git-sync** sidecar keeping the
repository fresh and read-only, plus model/state PVCs. Native sidecar on K8s ≥ 1.29, with a
classic-sidecar fallback for older clusters.

The pod binds `0.0.0.0`, so like the container it **fails closed**: set `auth.apiKeys` (recommended —
source them from a `Secret`, via `auth.existingSecret`) or `auth.allowUnauthenticated=true` for a trusted
network. A default `helm install` with both unset will not start. Since 4.0.0 each key must be at least 32
characters, or the pod refuses to start and crash-loops:

```bash
kubectl create secret generic sankshep-keys --from-literal=api-keys="$(openssl rand -hex 32)"
# then: --set auth.existingSecret=sankshep-keys
```

!!! warning "The `Host` allow-list — Ingress users will get 403 until they act"
    Since 2.0.0 the chart renders a `Host` allow-list by default, derived from **this release's own Service
    DNS** plus the loopback names a `kubectl port-forward` makes a browser send. That is the hardening a
    routable bind calls for (see [Security & privacy](security.md)), and before that setting it meant
    hand-writing `extraEnv`.

    An **Ingress hostname cannot be derived from anything the chart knows**, so it has to be declared:

    ```yaml
    allowedHosts:
      extra: ["sankshep.internal.example.com"]
    ```

    `allowedHosts.enabled: false` renders `SANKSHEP_ALLOWED_HOSTS=*` from 4.0.0 on — the server refuses a
    routable bind with no list, so "disabled" means any `Host` is accepted deliberately, with `Origin`
    validation still on. **Health probes are exempt either way**, so a 403 here never takes the pod down —
    and `NOTES.txt` prints the effective list after install, so nobody has to guess why a request was
    refused.

    Also in the chart since 2.0.0: the pod no longer mounts a Kubernetes API token it never used, the
    liveness probe gets 5 seconds instead of Kubernetes' 1-second default (this pod runs ONNX inference) —
    and from 4.0.0 so does the readiness probe — and an optional ingress-only `NetworkPolicy` is available —
    off by default, because a network policy is a decision about your cluster, not one a chart should make.
    No egress policy is templated: that would break git-sync and the model download.

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
2. **Mirror the image into your own registry, unmodified.** The Helm chart's `image.repository` exists to
   be repointed at it. Ship the model beside the image rather than inside it: mount a volume or host
   directory holding `bge-small-en-v1.5/model.onnx` and `bge-small-en-v1.5/vocab.txt` at `/models`, the
   image's `SANKSHEP_MODEL_DIR`, and set `SANKSHEP_MODEL_OFFLINE=1`. Do not build a derived image with
   the model baked in: the licence, the `LICENSE` file at `/app/LICENSE` in the image, does not permit
   derivative works of it.

On the Helm chart, set `model.offline: true` (which sets `SANKSHEP_MODEL_OFFLINE=1`) alongside a
pre-populated model PVC.

## Authentication

The HTTP surface is unauthenticated on the loopback default. On any **non-loopback bind** it is
**mandatory**: the server refuses to start unless you configure one of the modes below or set
`SANKSHEP_ALLOW_UNAUTHENTICATED=1` to deliberately accept an unauthenticated bind on a trusted,
network-isolated host. For a LAN or enterprise deployment:

- **API key:** `SANKSHEP_API_KEYS=<key>` (comma-separated for multiple keys; the singular
  `SANKSHEP_API_KEY` remains a single-key alias) — every request except the health probes needs
  `Authorization: Bearer <key>`. Since 4.0.0 every key must be at least 32 characters
  (`openssl rand -hex 32`), or the server refuses to start.
- **OAuth 2.1 Resource Server:** `SANKSHEP_OAUTH_AUTHORITY` + `SANKSHEP_OAUTH_AUDIENCE` — Sankshep
  validates tokens from *your* identity provider (audience/issuer/lifetime/signature + scopes) and
  serves Protected Resource Metadata. It is a Resource Server only; it never issues or forwards tokens.
  Set both or neither: since 4.0.0 one alone refuses to start.
- **DNS-rebinding:** `SANKSHEP_ALLOWED_HOSTS` pins the `Host` header. Since 4.0.0 it is **required** on a
  non-loopback bind — the names clients use, or `*` to accept any `Host` deliberately — and it does not
  replace authentication.

## Telemetry (opt-in, counts-only)

Metrics export is off by default (zero outbound). Point Sankshep at a **customer-controlled** OpenTelemetry
Collector to aggregate KPIs across a fleet:

```bash
export SANKSHEP_OTLP_ENDPOINT=http://otel-collector:4317
```

It exports **counts and histograms only** — never code content or file paths (the only per-metric repo
identifier is a folder name). Under `--http`, ASP.NET Core's own HTTP server metrics are exported with them.
Leave the endpoint unset for a fully local, zero-egress deployment.
