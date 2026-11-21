# Backups & Snapshots

Only for application data, k3s state itself can be rebuilt from git + NixOS config.

1. Encrypted, incremental backups of `/data/crypt` to Proton Drive to protect against disk failure.
2. ZFS snapshots of `/data/crypt` to protect against mistakes, bad updates, corruption...
3. Offline and offsite backups, manual, not documented here.

## Restic --> rclone --> Proton Drive

- restic: because rclone does not provide encryption and deduplication.
- rclone: like rsync but for cloud providers, including Proton Drive.

Restic does the backup stuff, rclone is the pipeline to Proton Drive.

## ZFS snapshots

Using NixOS ZFS auto-snapshotting service: https://search.nixos.org/options?channel=26.05&query=services.zfs.autoSnapshot

```nix
services.zfs = {
  autoSnapshot = {
    enable = true;
    frequent = 4;
    daily = 7;
    hourly = 24;
    weekly = 4;
    monthly = 3;
  };
  autoScrub = {
    enable = true;
    pools = [ "data" ];
  };
};
```

ZFS commands:

```sh
zfs list -t snapshot
zfs rollback data/crypt@zfs-auto-snap_frequent-2026-07-20-18h45
```

## Setup

1. Create the rclone config file (can be done locally). The cronjob expects `restic` as repo name.

```sh
docker run -it --rm -v rclone-cfg:/config rclone/rclone:latest config
docker run --rm -v rclone-cfg:/config rclone/rclone:latest ls proton:
docker run --rm -v rclone-cfg:/config --entrypoint /bin/sh rclone/rclone:latest \
  -c 'cat /config/rclone/rclone.conf' | sed 's/^/    /'
```

When rclone config asks for 2fa and otp_secret_key, leave 2fa empty and only fill in the base32 otp_secret_key. This lets the container generate its own codes.

2. Fill in the encrypted secrets.yaml file: `sops infrastructure/backup/secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
    name: backup
    namespace: backup
type: Opaque
stringData:
    rclone.conf: |
        [proton]
        type = protondrive
        username = REDACTED
        password = REDACTED
        otp_secret_key = REDACTED
        client_uid = REDACTED
        client_access_token = REDACTED
        client_refresh_token = REDACTED
        client_salted_key_pass = REDACTED
    RESTIC_PASSWORD: REDACTED
    RC_USER: REDACTED
    RC_PASS: REDACTED
```

3. Commit, push, reconcile

```sh
git add -A
git commit -m "backup: deploy restic & rclone to proton drive"
git push
flux reconcile source git flux-system        # re-clone git now
flux reconcile ks flux-system --with-source  # rebuild clusters/khazadum
flux reconcile ks backup --with-source       # apply manifests + decrypt the secret
```

4. Trigger first manual backup and verify

```sh
kubectl -n backup create job --from=cronjob/backup manual-test
kubectl -n backup logs job/manual-test -c restic -f
kubectl -n backup logs job/manual-test -c rclone -f
```

After the first backup, runs are incremental.

## Verify

```sh
kubectl -n backup get jobs
kubectl -n backup logs job/<latest> -c restic
```

## Alerting

See `BackupJobFailed` in `infrastructure/observability/kube-prometheus-stack/release.yaml`

## Cheat sheet

You can use the restic cli by running the restore-pod.

```sh
# start & enter the restore pod
kubectl -n backup apply -f infrastructure/backup/restore-pod.yaml
kubectl -n backup exec -it restore -c restic -- sh

# what backups exist
restic snapshots
restic snapshots latest
restic stats latest                 # deduped size
restic stats latest --mode raw-data # size on Proton

# what's inside a snapshot
restic ls latest                 # use with grep or less
restic diff 85c75e65 f7c4c75f    # changed files between snapshots
restic find 'nextcloud-*.sql.gz' # which snapshots contain certain files

# restore data
restic dump latest /data/nextcloud/dumps/nextcloud-2026-07-20.sql.gz > dump.sql.gz
restic restore latest --target /data/restored   # on host: /data/crypt/restored

# delete a backup
restic forget 85c75e65 --dry-run # preview
restic forget 85c75e65 --prune   # drop it

# troubleshooting
restic unlock                      # remove stale lock
restic check                       # tests the repository for errors
restic check --read-data-subset=5% # content sample

# remove the restore pod
kubectl -n backup delete pod restore
```