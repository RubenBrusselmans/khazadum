# Improvement roadmap

Findings from a full audit of the manifests, done 2026-11-20, ranked by how
much cluster improvement they buy per hour of work. Items are removed once
they are done, so everything here is still open. New findings get added at
the bottom; the numbering only changes when an item is removed.

What is deliberately not on this list: backup coverage itself (restic
covers all of `/data/crypt` with verification and retention), Flux
failure alerting to Matrix, digest pinning and Renovate updates, and the
CI setup. The audit found those in good shape; the one backup gap that
remains is the failure alert, item 3 below. Network policies are off the
list too: default-deny ingress is enforced in every namespace since
2026-11-21 and new apps are expected to ship their own policy (see
[deploying-apps.md](deploying-apps.md)). Egress filtering was evaluated
and deliberately skipped — the maintenance cost outweighs the benefit
for a single-operator cluster; LAN-side egress hardening can be
revisited if the threat model changes. Health probes went in on
2026-11-21, and the Recreate strategy gap closed in the same pass.
Deliberate exceptions: ansible is a `sleep infinity`
toolbox with nothing to probe, nextcloud gets no liveness (an httpGet
would restart the pod while maintenance mode is on), and the Forgejo
deployment lives in the external gundabad repo, so its probes are a
follow-up there. The remaining work is mostly about fair scheduling and
wiring pieces together.

## 1. Resource requests and limits

No container under `apps/` declares resource requests or limits, and the
same goes for woodpecker, git and the backup CronJob; only the
observability stack sets them. On a single node that means the kubelet
has no idea which workload matters.
One jellyfin transcode or a busy nextcloud cron run can push everything
else aside, including Flux, renovate and the metrics stack, while all
persistent data sits on a spinning disk that hates contention.

Start with requests on the heavy apps (jellyfin, nextcloud, keycloak,
openwebui) so scheduling uses realistic numbers. Add limits where a
runaway is plausible: the nextcloud cron sidecar, openwebui. Don't
bother with limits on the small static apps; requests are enough there.

The urgency is not theoretical: on 2026-11-21 the woodpecker server was
restarted by its liveness probe right after a pipeline push, with
`database is locked` SQLite errors and no limits to contain it.

## 2. Keycloak has no clients

Keycloak runs, it is built by CI with the khazadum theme, and nothing
authenticates against it. No app manifest mentions OIDC or OAuth.

Either give it clients or question why it runs at all. The easy wins are
wiki-js and Grafana: both support OIDC out of the box and would replace
their local logins. Nextcloud works too but needs the OIDC user backend
app. Each client also needs a realm, a client definition and a secret in
git, following docs/keycloak.md and docs/secrets.md.

## 3. The backup failure alert never resolves

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

## 4. TLS ends at the firewall

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

## 5. Small cleanups

A few things to batch whenever an app is touched anyway:

- App namespaces carry no PodSecurity labels, and most containers run as
  root. Fine for a single-operator lab, worth tightening if the cluster
  ever grows a second tenant.
- `privileged: true` on jellyfin and device-metrics-exporter is
  load-bearing for GPU device access. Leave it alone until item 6.

## 6. GPU device plugin

The proper fix for the privileged containers: an amdgpu device plugin
DaemonSet (`rocm/k8s-device-plugin`) advertising `amd.com/gpu`, so
jellyfin and device-metrics-exporter request the resource and drop
`privileged` for good. This replaces the piecemeal workarounds
documented in docs/jellyfin.md. Larger job, and partly dependent on what
upstream ships, so it sits at the bottom of the list.
