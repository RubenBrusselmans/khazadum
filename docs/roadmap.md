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

Forgejo, the nextcloud app/cron containers, the wiki-js postgres,
homepage, hermes and openwebui need a different route: they also
require root with full capabilities (forgejo and hermes are s6-overlay
images, nextcloud's entrypoint chowns and su's to www-data, wiki-js's
PGDATA subdir is not 999-accessible, homepage and openwebui write into
dirs a caps-stripped root cannot). Their hardening was reverted during
the 2026-11-27 attempt after crash-loops. De-rooting means a rootless
image variant and re-mapped data dir ownership.

The bigger unlock is a storage-model change: migrating app data from
hostPath to PVCs would lift the hostPath restriction and let most of
those namespaces move from `enforce: privileged` to `baseline`.
