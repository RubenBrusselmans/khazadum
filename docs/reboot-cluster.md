# Reboot cluster

Drain the node first (the `kubectl drain` command below evicts pods and
gives them time to shut down cleanly). Databases like mariadb and postgres,
plus prometheus, run crash recovery on the next boot if they were killed
uncleanly, and that recovery is slow. On an undrained node every pod starts
at once after the reboot and the concurrent startup overloads the HDD.

```sh
kubectl drain tower --ignore-daemonsets --delete-emptydir-data --force
sudo systemctl stop k3s
sudo k3s-killall.sh

sudo reboot

sudo zfs load-key data/crypt
sudo zfs mount data/crypt
mountpoint /data/crypt
sudo systemctl start k3s
kubectl uncordon tower
```

## Related

- [commands.md](commands.md) — status/verification commands
- [AGENTS.md](../AGENTS.md) — suspension pitfalls (`flux suspend` + removal orphans resources)
