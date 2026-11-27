# Improvement roadmap

Findings from a full audit of the manifests, done 2026-11-20, ranked by
how much cluster improvement they buy per hour of work. Items are removed
once they are done, so everything here is still open. New findings get
added at the bottom; the numbering only changes when an item is removed.

## 1. De-root the root-default containers

openwebui, the keycloak and wiki-js app containers, hermes, backup
(rclone/restic) and homepage still run as root: their images declare no
non-root USER and their hostPath data dirs are root-owned. Every pod
already has the rest of the hardening (no privilege escalation, all
capabilities dropped, seccomp), so the remaining step is a uid switch:
chown the host data dir to a fixed uid, set `runAsUser` + `runAsNonRoot`
per container, one app at a time with rollout verification.

Forgejo needs a different route: its image runs s6-overlay, which
requires root with full capabilities (hardened as 2026-11-27, it
crash-looped). De-rooting means switching to the forgejo rootless image
and remapping the data dir ownership.

The bigger unlock is a storage-model change: migrating app data from
hostPath to PVCs would lift the hostPath restriction and let most of
those namespaces move from `enforce: privileged` to `baseline`.
