# Nextcloud

Single-pod stack: mariadb + redis + nextcloud + cron + db-dump sidecar share
one pod (localhost networking between them). External URL: cloud.khazadum.net
(ingress.yaml).

## Constraints

- `strategy: Recreate` + `replicas: 1` is required: mariadb runs on
  a hostPath volume, and `RollingUpdate` deadlocks two pods on the same
  database files.
- mariadb is constrained to `^11` in `renovate.json` (nextcloud support
  matrix). Do not widen it casually: nextcloud major releases dictate the
  supported mariadb line.
- All images are digest-pinned and updated by Renovate
  ([renovate.md](renovate.md)). Never hand-edit digests.

## php-overrides (php-overrides.yaml)

ConfigMap mounted as `/usr/local/etc/php/conf.d/zz-session-override.ini`:

- Redis session locking disabled (`redis.session.locking_enabled = 0`):
  the stock image ships `locking_enabled=1` with `lock_retries=-1` and no
  lock_expire (infinite hold), so every parallel request of one user
  serializes on the Redis session lock. The symptom was Nextcloud feeling
  single-threaded per user.
- opcache + APCu headroom (AIO-style values): opcache 256M / 20000 files /
  64 interned strings buffer, APCu 128M + CLI enabled.

## Readiness probe

The probe execs into `config.php` and fails while `maintenance` mode is on.
A pod stays not-Ready after a failed upgrade instead of serving broken
pages. If the pod is not-Ready after an update, check maintenance mode first:

```sh
kubectl -n nextcloud logs deployment/nextcloud -c nextcloud | tail
```

Clear it via `occ` if stuck (exec into the nextcloud container).

## Database dumps (db-dump sidecar)

- Dumps daily at 02:30 (before the 03:00 restic backup), gzip to
  `/data/crypt/nextcloud/dumps` (`nextcloud-YYYY-MM-DD.sql.gz`), keeps the
  14 most recent.
- The dumps ride the encrypted backup to Proton Drive
  ([backup.md](backup.md)).
- Restore: restic restore the dump, then `mariadb` client import (use the
  restore pod from [backup.md](backup.md)).

## Secrets

`secret.yaml` (SOPS): `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`. Verify by
DATA-key count only. Values are operator-confidential.

## Data on the host

`/data/crypt/nextcloud/{mariadb,redis,nextcloud,dumps}` plus the shared
`/data/crypt/media` volume (same media jellyfin serves, mounted read-write
here at `/media`).
