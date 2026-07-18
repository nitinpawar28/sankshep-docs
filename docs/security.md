# Security & privacy

Sankshep is built to run entirely on your machine. This page states exactly what it stores, where, and what —
if anything — leaves the host, and the controls that back those claims.

## Data handling: what leaves your machine

**By default, nothing.** No code, file paths, queries, or memory content is sent anywhere. Embeddings are
computed locally via ONNX Runtime; vector search is local via sqlite-vec.

The **only** outbound network activity is:

| Activity | When | What is sent |
|---|---|---|
| Embedding-model download | First `index_repo`/`search_code`, unless side-loaded | A checksum-verified request for the public model files — **no repo content**. Suppress with `SANKSHEP_MODEL_OFFLINE=1` + `SANKSHEP_MODEL_DIR`. |
| Telemetry export | **Only** if you set `SANKSHEP_OTLP_ENDPOINT` | Counts and a low-cardinality repo tag (the folder name) — **never** file paths, queries, or code. To a collector **you** run. |

There is **no telemetry by default** and no dollar/billing phone-home — Sankshep is never told which model you
use or what you pay (ADR-0011, ADR-0017). No repository code or content ever leaves the machine; the model
download is a one-time inbound fetch of public files, and it is disableable for air-gapped use.

!!! note "`SANKSHEP_PROMETHEUS=1` is a pull endpoint, not outbound"
    In HTTP mode it exposes a local `/metrics` endpoint that **your** Prometheus scrapes — the server never
    initiates that connection, so it is inbound and not listed above. Only `SANKSHEP_OTLP_ENDPOINT` (OTLP
    push) sends telemetry outbound.

!!! success "Proven, not just asserted"
    A [continuous-integration air-gap job](#hardening-verified-posture) runs the server on a **no-egress
    network** and confirms `summarize_repo`, `index_repo`, and `search_code` all work with outbound traffic
    physically blocked — and a socket-level test asserts `get_context` opens **zero** outbound connections.

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
stdio mode has no network surface at all.

**The HTTP server fails closed.** If it is bound to a non-loopback interface (a container/Kubernetes
`ASPNETCORE_URLS=0.0.0.0`) with authentication set to `None`, it **refuses to start** — you must either
configure authentication or explicitly accept the exposure:

- **API-key auth** (recommended for LAN/cluster): `SANKSHEP_API_KEYS` (bearer keys, compared in constant time).
- **OAuth 2.1** resource-server mode: `SANKSHEP_OAUTH_*`.
- **Explicit opt-in** for a trusted, network-isolated deployment: `SANKSHEP_ALLOW_UNAUTHENTICATED=1`.

Health probes stay anonymous. The Helm chart is **secure by default**: it surfaces an `auth` block and will
not serve unauthenticated on `0.0.0.0` unless you opt in.

## Hardening & verified posture

These controls are enforced in code and covered by tests/CI:

| Area | Control |
|---|---|
| **Egress** | No repo content leaves the host; socket-level test + no-egress air-gap CI prove it. |
| **Telemetry** | Off by default; opt-in only; counts + folder-name tag, never paths/queries/code. |
| **Path scope** | Relative tool paths are confined to the served `--repo` root — a `../..` escape is rejected. |
| **SQL** | All SQLite access (facts, vectors, stats) uses bound parameters — no string-built queries. |
| **Deserialization** | Typed YAML/JSON only; no polymorphic type resolution, no `BinaryFormatter`. |
| **Untrusted repos** | `.gitignore` matching is ReDoS-safe (no catastrophic backtracking); `.docx`/`.pdf` extraction is size-capped against decompression bombs; directory walks are iterative and symlink-cycle-safe. |
| **Auth** | Fails closed on unauthenticated non-loopback binds; API keys compared with a constant-time equality. |
| **Secrets** | API keys / tokens are never written to logs; all logging is paths/errors/counts only. |
| **Isolation** | Container runs non-root, read-only root filesystem, all Linux capabilities dropped. |

## Independent security review

Sankshep undergoes **adversarial security review** — findings are refuted before they're accepted, and
confirmed issues are fixed before release. The most recent review (**v1.8.0**) audited ten dimensions
(path traversal, injection, deserialization, egress, secrets, tool-input abuse, HTTP-tier auth,
supply-chain, denial-of-service, and command/crypto misuse) and found **no critical or high-severity
vulnerabilities**. The medium/low hardening items it surfaced — the fail-closed auth bind, ReDoS-safe
ignore matching, relative-path containment, document size limits, and cycle-safe enumeration above — are
remediated as of **v1.8.0**.

## Supply chain

The NuGet package is published with **OIDC Trusted Publishing** — GitHub's OIDC token is exchanged for a
short-lived NuGet key at release time, so **no long-lived API key is stored** anywhere. NuGet audit is
enforced at build time: the real vulnerability codes (**NU1901–NU1904**) are treated as build **errors**, so
a flagged advisory fails the build — that's how the OpenTelemetry 1.10.0 CVE was caught.

Container images are published to GHCR, run as a non-root user with a read-only root filesystem and all Linux
capabilities dropped, and pin their base image by minor tag. The embedding-model download is SHA-256 verified
and fail-closed. For the strictest posture, pin the image by digest and pre-bake or side-load the model.

## Reporting a vulnerability

Please report suspected security issues **privately** — do **not** open a public issue, which would disclose
the problem before a fix exists.

[**→ Report a vulnerability privately**](https://github.com/nitinpawar28/sankshep-docs/security/advisories/new)

This opens a GitHub private security advisory visible only to the maintainer, and you'll get a response there.
(Sankshep's product source is a private repository, so vulnerability reports are received through this public
project's advisory channel.) See also [`SECURITY.md`](https://github.com/nitinpawar28/sankshep-docs/security/policy).
