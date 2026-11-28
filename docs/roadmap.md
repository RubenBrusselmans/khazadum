# Improvement roadmap

Findings from a full audit of the manifests, done 2026-11-20, ranked by
how much cluster improvement they buy per hour of work. Items are removed
once they are done, so everything here is still open. New findings get
added at the bottom; the numbering only changes when an item is removed.

## 1. Nextcloud de-root: research

The official image is root-orchestrated: the entrypoint chowns, creates
lock files as root and su's down to www-data, and crond needs
CAP_SETGID. De-rooting means the fpm-alpine variant plus a custom
entrypoint, or accepting root. Investigate what the nextcloud
ecosystem (the `nextcloud/all-in-one` approach, community Helm charts)
does before designing anything.

## 2. Hermes de-root: research

The hermes-agent image is s6-overlay based with no published rootless
variant. Options: request one upstream, build a wrapper image, or
accept root. Low priority — single internal-facing workload, digests
pinned, network default-deny.

## 3. Migrate app data from hostPath to PVCs

The storage-model change that lifts the hostPath blocker: PSA baseline
forbids hostPath volumes, so the eleven `enforce: privileged`
namespaces cannot be lowered until their data moves to PVCs (k3s
local-path provisioner would do). Biggest scope of all the items —
worth splitting per-app once started — but it unlocks `baseline` for
most of the fleet and makes the single-node storage story inspectable
via kubectl instead of ssh.

## 4. Restic incremental backups via fixed pod hostname

Every backup job pod gets a random hostname, so `restic backup` never
finds a parent snapshot and re-reads the full ~425k-file tree nightly
(observed live during the 2026-11-28 caps test; a full run takes
~42 min). Setting
a fixed `hostname: backup` on the job pod template enables parent
snapshot selection: nightly runs then read only changed files instead
of everything. Retention already groups by paths, so it stays correct;
the first run after the change is still a full read.
