# Improvement roadmap

Findings from a full audit of the manifests, done 2026-11-20, ranked by
how much cluster improvement they buy per hour of work. Items are removed
once they are done, so everything here is still open. New findings get
added at the bottom; the numbering only changes when an item is removed.

## 1. Hermes de-root: research

The hermes-agent image is s6-overlay based with no published rootless
variant. Options: request one upstream, build a wrapper image, or
accept root. Low priority — single internal-facing workload, digests
pinned, network default-deny.

## 2. Migrate app data from hostPath to PVCs

The storage-model change that lifts the hostPath blocker: PSA baseline
forbids hostPath volumes, so the eleven `enforce: privileged`
namespaces cannot be lowered until their data moves to PVCs (k3s
local-path provisioner would do). Biggest scope of all the items —
worth splitting per-app once started — but it unlocks `baseline` for
most of the fleet and makes the single-node storage story inspectable
via kubectl instead of ssh.

## 3. Restic incremental backups via fixed pod hostname

Every backup job pod gets a random hostname, so `restic backup` never
finds a parent snapshot and re-reads the full ~425k-file tree nightly
(observed live during the 2026-11-28 caps test; a full run takes
~42 min). Setting
a fixed `hostname: backup` on the job pod template enables parent
snapshot selection: nightly runs then read only changed files instead
of everything. Retention already groups by paths, so it stays correct;
the first run after the change is still a full read.

## 4. Nextcloud de-root: implement

Design and findings in [nextcloud-deroot.md](nextcloud-deroot.md)
(research 2026-11-28): run the stack as uid 33 via the debian fpm image
plus an nginx sidecar, replace the crond sidecar with a loop, drop
`REDIS_HOST` and move the session config into php-overrides, db-dump to
999, one-time chown of the data tree. Needs operator host work (chown,
digest resolve) before the manifest push, and a reconcile with
verification — four other Kustomizations depend on nextcloud being
Ready.
