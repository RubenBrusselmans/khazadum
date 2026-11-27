# Security

## Privileged GPU pods: accepted

Two pods run `privileged: true`, both for GPU access on the single
Radeon RX 7600 XT:

- jellyfin for VAAPI transcoding
- device-metrics-exporter for GPU metrics

Why nothing weaker works: the devices cgroup controls which device
files a container may open, and a hostPath mount does not put its
device nodes on that allowlist. Every `open()` fails with EPERM until
the pod is privileged. Three de-privileging attempts on jellyfin
(reverted 2026-08-22) hit this wall; the history is in
[jellyfin.md](jellyfin.md).

The real fix would be the amdgpu device plugin
(`rocm/k8s-device-plugin`) advertising `amd.com/gpu`: pods request the
resource, and the kubelet adds the device nodes to the allowlist at
container start. Considered 2026-11, rejected:

- It would de-privilege exactly one pod. The exporter can never use
  the resource: allocation is exclusive, the GPU is one
  unpartitionable unit (no MIG equivalent on consumer RDNA), and its
  metrics need privileged `/dev/kfd` access regardless. Jellyfin would
  be the only consumer.
- The cost is a permanent infra component: namespace, DaemonSet, KS,
  Renovate pinning, a plugin pod on the node forever.
- Nothing else on offer was better. The Helm chart ships a stale
  image, the official DRA driver (`k8s-gpu-dra-driver`) is a young
  v1.0.x, CDI is not reachable from a pod spec, and generic plugins
  are unofficial projects.

One pod cleaner, for that much permanent machinery, is not a good
trade on a homelab. Both pods stay privileged. Do not strip it
piecemeal.

What bounds the risk today: both namespaces default-deny ingress
(jellyfin answers only traefik on 8096; the exporter answers only
inside observability, its port 5000 is never exposed through
Traefik), both images are digest-pinned, and both pods are
`replicas: 1` with `strategy: Recreate`.

Revisit if untrusted tenants or workloads land here, if the GPU
becomes partitionable or a second node adds one, or if the exporter
drops the `/dev/kfd` metrics.

## Pod hardening

Every raw-manifest container carries an explicit securityContext:
`allowPrivilegeEscalation: false`, `capabilities: drop: [ALL]`, and a
pod-level `seccompProfile: RuntimeDefault`. `runAsNonRoot` (with an
explicit `runAsUser`) is declared only for images that already run as
that uid — postgres (999), mariadb/redis (999), woodpecker (1000),
awx-ee (1000), renovate (12021), matrix-receiver (nonroot). Root-default
images (openwebui, keycloak, wiki, hermes, backup rclone/restic,
homepage) keep their uid because their hostPath data dirs are
root-owned; de-rooting them is on the [roadmap](roadmap.md). Forgejo
carries no hardening at all: its image uses s6-overlay, which must run
as root with full capabilities to supervise services and drop to the
git user itself (a hardened attempt on 2026-11-27 crash-looped the
pod: s6-svscan could not open its lock as non-root).

Namespace PSA labels mirror this: `enforce: restricted` on ansible,
cert-manager and renovate; explicit `enforce: privileged` on the
hostPath-bound namespaces (backup, git, hermes, homepage, jellyfin,
keycloak, nextcloud, observability, openwebui, wiki-js, woodpecker) —
PSA baseline forbids hostPath volumes, so those cannot be lowered until
data moves off hostPath. jellyfin and device-metrics-exporter also run
`privileged: true` by decision (GPU device access, see below). The
gundabad namespace is managed in the gundabad repo and is not labeled
from here.

## Network policies

Every raw-manifest namespace ships a default-deny ingress
NetworkPolicy plus a narrow traefik allow for its app
(`apps/*/networkpolicy.yaml`, `infrastructure/*/networkpolicy.yaml`).
cert-manager and renovate are chart-managed and ship none; chart
defaults plus Traefik is acceptable for now and is tracked in
[roadmap.md](roadmap.md).

## Pointers

- Secrets at rest, key rotation: [secrets.md](secrets.md)
- Internal PKI, TLS trust: [cert-manager.md](cert-manager.md)
- Supply chain, digest pinning: [renovate.md](renovate.md)
- Hardening backlog: [roadmap.md](roadmap.md)