# Nextcloud de-root

Research for the roadmap item, done 2026-11-28. Decides how the `nextcloud`
and `cron` containers can stop running as root — or whether root stays as a
documented exception, like the GPU pods ([security.md](security.md)).
Everything upstream here was verified against the current sources:
`nextcloud/docker` (docker-entrypoint.sh, docker-cron.sh, Dockerfile
templates), `nextcloud/all-in-one` (Containers/nextcloud), and
`nextcloud/helm` (values.yaml, templates/deployment.yaml).

## Current state

Single pod, five containers ([nextcloud.md](nextcloud.md)):

- mariadb and redis: already de-rooted — `runAsUser: 999`,
  `runAsNonRoot: true`, drop ALL.
- nextcloud (apache variant) and cron: no container securityContext at all;
  root is what makes the stock entrypoint work and what busybox crond
  requires.
- db-dump: root with drop ALL; writes the daily dumps.

Host data: `/data/crypt/nextcloud/{mariadb,redis,nextcloud,dumps}` plus the
shared `/data/crypt/media` (jellyfin reads it as PUID 1001/PGID 100). The
app serves three hostnames through Traefik and holds the cluster's main
user-data tree, which is what makes removing root here worthwhile.

## Upstream findings

### The official image supports non-root runs

The README has a dedicated section ("Running as an arbitrary user / file
permissions / changing the default container user"), and the entrypoint
branches on uid:

- as root: computes user/group (www-data), rsyncs the install tree with
  `--chown`, and `run_as()` uses `su`.
- as non-root: `run_as()` is plain `sh -c` (no su), rsync drops `--chown`,
  and everything runs under the container uid. The init-sync lock file
  (`nextcloud-init-sync.lock`) lives in the html volume, so no host-level
  lock files and no CAP_SETGID.

The roadmap's premise ("entrypoint chowns, creates lock files as root,
su's down, crond needs CAP_SETGID") describes only the root path. Non-root
runs expect the volume already owned by the container uid — a one-time
operator chown.

uid map: debian images (`apache`, `fpm`) have www-data = 33; alpine images
(`fpm-alpine`) have www-data = 82.

### Background jobs: crond is the real blocker

`/cron.sh` is now `docker-cron.sh` upstream: `exec busybox crond -f -L
/dev/stdout`, driving the image-shipped crontab
`*/5 * * * * php -f /var/www/html/cron.php` as www-data. busybox crond
cannot run as non-root (the official helm chart documents this verbatim:
"crond does not work when not running as root"). Non-root replacements:

- a Kubernetes CronJob running `php -f /var/www/html/cron.php` as the same
  uid — the chart's own answer ("Should run as same user as the Nextcloud
  pod");
- an in-pod sidecar loop (`while :; do php -f /var/www/html/cron.php;
  sleep 300; done`) — replicates the crontab line exactly and keeps the
  single-pod model. Preferred here for the smaller diff; the cron
  container's command override already skips entrypoint init, and the loop
  keeps it that way.

### Hidden blocker: the entrypoint writes into conf.d

With `REDIS_HOST` set, the entrypoint unconditionally writes
`/usr/local/etc/php/conf.d/redis-session.ini`. That image path is
root-owned, so as uid 33 the write fails and `set -eu` kills the container
(CrashLoop). The official helm chart works around exactly this with an
initContainer whose only job is to `touch` the file on an emptyDir (its
comment: "fix permission denied error"). Cleaner for us: drop the
`REDIS_HOST` env — config.php already carries the redis config, the env
only matters for fresh installs — and ship the session settings in our own
php-overrides ConfigMap, already mounted read-only into conf.d
([nextcloud.md](nextcloud.md)).

### AIO is not rootless — but it picks uid 33

`nextcloud/all-in-one`'s nextcloud container is a custom fpm-alpine build:
it remaps alpine's www-data (82) to uid/gid 33 for compatibility with
debian-era data, sets `www-data:root` with `chmod g=u` on the web tree, and
still starts as root (`USER root:root`, dinit supervisor). Its start.sh
does the root work (package handling, GPU group, conditional chown behind
a touch-probe), then `su-exec www-data /entrypoint.sh` — running the
official entrypoint as uid 33 non-root. The webserver is a separate
container. Lesson: even the flagship keeps a root wrapper. The ecosystem
consensus is uid 33 everywhere and group-readable data; nobody runs
nextcloud fully rootless.

### The official helm chart: reference, not a migration target

`nextcloud/helm` supports `runAsUser: 33` on the apache flavor (its pod
securityContext defaults to `fsGroup: 33`), runs cron as a root crond
sidecar or a non-root Kubernetes CronJob, and has an fpm+nginx mode with a
generated nginx config. It does not de-root by default — the container
securityContext defaults to `{}` (root). Switching would mean: splitting
mariadb/redis out of the pod (the chart has no sidecar-DB concept; its
subcharts are bitnami *legacy* images and our DBs are 127.0.0.1
sidecars), re-expressing php-overrides, the db-dump sidecar, the
maintenance-mode exec probe and the media hostPath through values, plus an
HTTP-only chart source (chart-releaser, no OCI mirror) against
[renovate.md](renovate.md) preferences — with four other Flux
Kustomizations depending on nextcloud being Ready. Zero de-root gain. The
chart stays a pattern reference only.

## Options

### A. Apache image as uid 33 (minimal diff)

Keep `nextcloud:35.0.3@<digest>` (apache variant); set the container
securityContext to `runAsUser: 33`, `runAsNonRoot: true`, drop ALL and add
back exactly `NET_BIND_SERVICE` (apache must bind :80 — service and
NetworkPolicy stay untouched).

apache-as-non-root needs k8s-side fixes because its image paths are
root-owned: an emptyDir over `/var/run/apache2` (pidfile dir), and log
handling — `/var/log/apache2` holds root-owned symlinks to
/dev/stdout|/dev/stderr; apache can open the existing symlinks, but any
enabled config that *creates* a log file there (debian's
other-vhosts-accesslog) must be neutralized by mounting an empty file over
it. Which confs need this is a verify-at-implementation item.

Note the chart's own `runAsUser: 33` example works differently: it leaves
the default capability set in place (which quietly includes
DAC_OVERRIDE), so apache can bypass the root-owned dirs — a half de-root
that still lets uid 33 override file permissions on every mount.

Everything else is shared with B: session-ini fix, cron replacement,
db-dump → 999, one-time chown.

Residual: one capability on the user-data container, and apache's root-era
paths papered over with mounts instead of removed.

### B. fpm + nginx sidecar (recommended)

Switch the app container to `nextcloud:35.0.3-fpm` — debian base, so
www-data is natively 33, no alpine uid churn — and add an nginx sidecar
serving statics from the same `/var/www/html` hostPath (read-only) and
proxying FastCGI to 127.0.0.1:9000:

- nginx (`nginx:alpine`, digest-pinned) as uid 33, config as ConfigMap:
  pid and temp paths under /tmp, logs to stdout/stderr, the nextcloud
  nginx rules from the official docs (the chart's generated default is the
  reference).
- Service targetPort 80 → 8080 and the traefik NetworkPolicy port
  likewise. php-fpm's 9000 is intra-pod loopback; NetworkPolicy never sees
  it.
- No capabilities anywhere; drop ALL on every container.

This is the shape AIO and the chart both converge on. It removes apache's
root-isms instead of papering over them, and the nginx ConfigMap is ours
to own like php-overrides. Cost: one new image pin, one new ConfigMap,
port plumbing in service + networkpolicy.

Shared with A:

- Unset `REDIS_HOST` on the app container; move the redis session config
  (save_path 127.0.0.1:6379, locking disabled) into the php-overrides
  ConfigMap.
- Replace the cron sidecar's `/cron.sh` with the uid-33 loop.
- db-dump: `runAsUser: 999` + `runAsNonRoot: true` (the mariadb image's
  own uid; only the client binary runs), dumps dir chowned to 999 so
  rotation's `rm` keeps working.
- `runAsNonRoot: true` declared per container (repo convention), not at
  pod level.
- Keep uid 33, not alpine 82: no chown of `/media` (jellyfin reads it),
  matches AIO's remap choice. fsGroup stays unused, matching the repo's
  explicit-chown precedent (forgejo, keycloak, wiki-js).
- The readiness probe (exec php on config.php) keeps working: php and the
  config tree are readable as 33.

### C. Accept root

Document like the GPU decision and keep the roadmap entry closed. Cheapest,
but this is the main user-data workload; root here is the least hardening
per unit of exposure. Only if A and B both fail in practice.

## Decision

Recommendation: B. The doc stays open for the operator's call; A is the
low-effort fallback if the apache path (or B's nginx config) misbehaves in
practice.

Decision: B — operator, 2026-11-28.

## Migration sketch (for the implementation pass)

1. Operator, on the node: verify ownership first
   (`stat -c '%u:%g %n' /data/crypt/nextcloud/*` — expect 33 on the
   nextcloud tree, 999 on mariadb/redis, root on dumps), then one-time:
   `chown -R 33:33 /data/crypt/nextcloud/nextcloud` and
   `chown -R 999:999 /data/crypt/nextcloud/dumps` (old dumps are
   root-owned; as 999 the rotation `rm` would silently fail forever
   otherwise). `/media` untouched.
2. Resolve digests read-only:
   `docker buildx imagetools inspect nextcloud:35.0.3-fpm` and the chosen
   nginx tag; pin in git. Renovate keeps managing them afterwards.
3. Manifest edits per the chosen option; push;
   `flux reconcile ks nextcloud --with-source`.
4. Verify: pods 1/1 Ready; `id -u` is 33 in nextcloud/cron (999 in
   db-dump); web UI and desktop sync clients work; a test upload lands in
   `/data/crypt/media` and stays readable by jellyfin; occ via exec works;
   the next nightly backup succeeds (the BackupNotSuccessful alert is the
   backstop).
5. Rollback = git revert + reconcile; the chowns need no undoing (the root
   entrypoint re-chowns on the next image upgrade anyway).

Also during implementation: refresh the stale `renovate.json` description
("Nextcloud 34 MariaDB <= 11.8" → NC 35; the current matrix allows
MariaDB 10.11/11.4/11.8/12.3, so `^11` stays sensible).
