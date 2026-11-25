# Improvement roadmap

Findings from a full audit of the manifests, done 2026-11-20, ranked by how
much cluster improvement they buy per hour of work. Items are removed once
they are done, so everything here is still open. New findings get added at
the bottom; the numbering only changes when an item is removed.

## 1. GPU device plugin

The proper fix for the privileged containers: an amdgpu device plugin
DaemonSet (`rocm/k8s-device-plugin`) advertising `amd.com/gpu`, so
jellyfin and device-metrics-exporter request the resource and drop
`privileged` for good. This replaces the piecemeal workarounds
documented in docs/jellyfin.md. Larger job, and partly dependent on what
upstream ships, but it is the prerequisite for hardening the GPU
workloads, so it sits first.

## 2. Pod security hardening, one namespace at a time

Take each workload to the strongest PodSecurity level it tolerates:
restricted where possible, baseline otherwise, privileged where hostPath
demands it. Tightening is two layers: an explicit per-pod securityContext
(non-root user where the image allows, seccompProfile RuntimeDefault,
allowPrivilegeEscalation: false, capabilities drop ALL), and a namespace
PodSecurity label at the level its workloads pass.

Workflow per namespace: probe the target level first — `kubectl run
--dry-run=server` a Pod with the candidate spec in a scratch namespace
carrying the label; the admission verdict comes without creating
anything. Dry-run checks admission only, not runtime writability
(writable cache dirs are the classic gap). Then edit git, reconcile, and
watch the rollout; a label change alone never restarts running pods.

- Today only ansible, cert-manager and renovate can reach restricted: the
  /data/crypt hostPath convention caps the other 11 at privileged, since
  PSA baseline and restricted both forbid hostPath volumes (backup, git,
  hermes, homepage, jellyfin, keycloak, nextcloud, observability,
  openwebui, wiki-js, woodpecker). Pod hardening still applies under a
  privileged label. Migrating app data to PVCs would unlock baseline for
  those 11 — a storage-model change, its own decision.
- cert-manager is already fully restricted-compliant via chart defaults,
  so its batch is the label alone. Known wrinkles: awx-ee only sleeps;
  renovate's cronjob needs a writable cache dir when non-root.
- `privileged: true` on jellyfin and device-metrics-exporter is
  load-bearing for GPU device access. Leave it alone until item 1.
