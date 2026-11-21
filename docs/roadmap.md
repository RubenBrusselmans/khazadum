# Improvement roadmap

Findings from a full audit of the manifests, done 2026-11-20, ranked by how
much cluster improvement they buy per hour of work. Items are removed once
they are done, so everything here is still open. New findings get added at
the bottom; the numbering only changes when an item is removed.

## 1. The backup failure alert never resolves

Backup failure alerting exists: the `BackupJobFailed` PrometheusRule in
infrastructure/observability/kube-prometheus-stack/release.yaml pages
matrix-receiver, and since `restic check` runs inside the same CronJob,
a non-zero exit covers all three failure modes (rclone cannot reach
Proton Drive, restic fails mid-run, the check finds corruption).

The flaw is the expression. It fires on
`kube_job_status_failed{namespace="backup"} > 0`, but failed Job objects
linger in history (`failedJobsHistoryLimit: 3`), so one bad night keeps
the gauge above zero forever: the alert re-pages every 12 hours and
never resolves, even after the problem is fixed and later runs succeed.

Replace it with a staleness check on the last successful run: alert when
the newest backup job with `kube_job_status_succeeded == 1` has a
`kube_job_status_completion_time` more than ~26 hours old (the job runs
nightly, so this catches one missed night with margin). The Flux alert
route does not help here: it only fires when the CronJob spec fails to
reconcile, not when a healthy spec produces a failed run.

## 2. TLS ends at the firewall

Caddy on OPNsense terminates TLS for the khazadum.net domain and
forwards plain HTTP to the cluster. Between OPNsense and Traefik, and
everywhere inside the node, traffic is unencrypted. Both ends sit on the
same machine, so this is not an emergency, but it means any pod with
network access can read all of it. docs/architecture.md already notes
the plan to close this.

The shape of the fix: get certificates to Traefik (cert-manager with a
DNS challenge through OPNsense, or reuse the certs Caddy already
fetches), then re-encrypt the Caddy to Traefik hop. Roughly a weekend,
most of it testing the ingress chain.

## 3. Small cleanups

A few things to batch whenever an app is touched anyway:

- App namespaces carry no PodSecurity labels, and most containers run as
  root. Fine for a single-operator lab, worth tightening if the cluster
  ever grows a second tenant.
- `privileged: true` on jellyfin and device-metrics-exporter is
  load-bearing for GPU device access. Leave it alone until item 4.

## 4. GPU device plugin

The proper fix for the privileged containers: an amdgpu device plugin
DaemonSet (`rocm/k8s-device-plugin`) advertising `amd.com/gpu`, so
jellyfin and device-metrics-exporter request the resource and drop
`privileged` for good. This replaces the piecemeal workarounds
documented in docs/jellyfin.md. Larger job, and partly dependent on what
upstream ships, so it sits at the bottom of the list.
