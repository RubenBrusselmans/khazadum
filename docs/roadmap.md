# Improvement roadmap

Findings from a full audit of the manifests, done 2026-11-20, ranked by
how much cluster improvement they buy per hour of work. Items are removed
once they are done, so everything here is still open. New findings get
added at the bottom; the numbering only changes when an item is removed.

## 1. Switch forgejo to its rootless image

The rootful forgejo image is s6-overlay based and hardening broke it
(2026-11-27, crash-loop). Forgejo publishes a `-rootless` variant that
runs as uid 1000 without s6: swap the image reference (one new digest
for Renovate to pin), chown `/data/crypt/git` to 1000, re-add the
caps-drop hardening. Biggest single win — the git KS is Flux's own
source, so it deserves the fastest de-root on the cluster.

## 2. Label the gundabad namespace

The gundabad manifests (and its namespace) live in the external
gundabad repo, so it was left out of the PSA labeling pass. Add the
`pod-security.kubernetes.io/enforce` label there at the level its
workloads pass, mirroring [security.md](security.md).

## 3. Backup restic: accept root or move it off-cluster

restic must read every file under /data/crypt, including root-owned
0600 files — a caps-stripped root silently skips them (found live in
the 2026-11-27 test run). Chowning the whole tree to one uid is not an
option. Either document root with full caps as an accepted exception
(like the GPU pods in [security.md](security.md)) or redesign the
backup as a host-level systemd job outside k8s. Decide, then record it.

## 4. Nextcloud de-root: research

The official image is root-orchestrated: the entrypoint chowns, creates
lock files as root and su's down to www-data, and crond needs
CAP_SETGID. De-rooting means the fpm-alpine variant plus a custom
entrypoint, or accepting root. Investigate what the nextcloud
ecosystem (the `nextcloud/all-in-one` approach, community Helm charts)
does before designing anything.

## 5. Hermes de-root: research

The hermes-agent image is s6-overlay based with no published rootless
variant. Options: request one upstream, build a wrapper image, or
accept root. Low priority — single internal-facing workload, digests
pinned, network default-deny.

## 6. Migrate app data from hostPath to PVCs

The storage-model change that lifts the hostPath blocker: PSA baseline
forbids hostPath volumes, so the eleven `enforce: privileged`
namespaces cannot be lowered until their data moves to PVCs (k3s
local-path provisioner would do). Biggest scope of all the items —
worth splitting per-app once started — but it unlocks `baseline` for
most of the fleet and makes the single-node storage story inspectable
via kubectl instead of ssh.
