# Improvement roadmap

Findings from a full audit of the manifests, done 2026-11-20, ranked by how
much cluster improvement they buy per hour of work. Items are removed once
they are done, so everything here is still open. New findings get added at
the bottom; the numbering only changes when an item is removed.

What is deliberately not on this list: backup coverage itself (restic
covers all of `/data/crypt` with verification and retention), Flux
failure alerting to Matrix, digest pinning and Renovate updates, and the
CI setup. The audit found those in good shape; the one backup gap that
remains is the failure alert, item 6 below. The remaining work is mostly
about blast radius (what one bad pod can reach), honest rollouts, and
wiring pieces together.

## 1. Network policies

Progress: default-deny ingress is live for nextcloud and keycloak
(2026-11-21, ingress-only; egress lockdown is a separate later pass).
Everything else is still open: any pod can open a connection to any
service anywhere.

Two corrections to the original audit wording, kept here so the threat
model stays honest:

- The nextcloud mariadb/redis and keycloak postgres are sidecars inside
  their app's pod and have no Services of their own. Nothing can reach
  them by DNS name; the only path is scanning the flat flannel pod
  network for the DB port on the pod IP, which a default-deny policy
  blocks. Intra-pod 127.0.0.1 traffic is unreachable from outside the
  pod and is not filtered by NetworkPolicy at all — no rules needed for
  the DB hop.
- The woodpecker gRPC is reachable cluster-wide, but registering as an
  agent requires the agent secret, so bare reachability is
  DoS/exploit surface, not instant access. The real untrusted tenant is
  the CI job pods in the woodpecker namespace (arbitrary build code).

The east-west surface that actually matters for the remaining
namespaces: the web Services (forgejo, woodpecker, grafana, prometheus,
loki, the apps in the external gundabad repo), matrix-receiver (any pod
can POST fake alerts into Matrix), and unrestricted egress everywhere.
Note the gundabad apps live in the other repo, so covering them is a
change there, not here.

Enforcement is confirmed working: k3s runs the default flannel +
embedded network-policy controller (no `--disable-network-policy`), and
a 2026-11-21 functional check from a pod proved a flux-allowed port
answers while a blocked port times out. The backup and flux-system
default-deny policies have been enforcing since 2026-08.

Next coverage, cheapest first: observability (matrix-receiver),
woodpecker, git, then gundabad, then egress.

Status update 2026-11-21: nextcloud, keycloak, observability and
woodpecker are done (same shape: default-deny + traefik allow;
observability also allows its own namespace port-agnostic for the scrape
mesh, woodpecker allows same-ns to the server's gRPC 9000). git is done
too: one allow policy lists its four legitimate clients on port 3000
(traefik, flux-system, renovate, woodpecker). All verified live:
ingress URLs answer, all 14 prometheus targets scrape, loki ingests
alloy's logs, a woodpecker pipeline ran clean, and renovate's manual
run succeeded. One upstream quirk surfaced: kube-router syncs its
allow-set asynchronously, so a CI pod that fires its first registry
request seconds after start can be rejected transiently; the kaniko
permission check is now skipped in the pipeline (kaniko documents this
exact flag for delayed network policies). Side finding from the same
session: the woodpecker server restarted on a liveness probe failure
right after a pipeline push, with `database is locked` SQLite errors —
no resource limits on it yet, so item 3 below got more urgent.

Remaining coverage: egress only. As of 2026-11-21 ingress is locked
down in every namespace, including the gundabad apps and jellyfin
(verified: all public hosts answer, pod-level probes show every
protected namespace denying cross-namespace connections, the gundabad
pod runs 10/10). No allow rule exists for anything to reach jellyfin
except traefik — none of the external apps currently call it; if one
gets wired up later, its namespace needs an explicit allow on 8096.
All Kustomizations Ready.

## 2. Health probes

Only keycloak, wiki-js, nextcloud (one probe) and woodpecker have any.
hermes, homepage, openwebui, jellyfin, git and ansible run with none.

The Flux Kustomizations deploy with `wait: true`, so Flux reports an
app as ready as soon as its pods start. Without a readinessProbe, a web
app that hangs during startup, or wedges hours later, still looks
healthy and nothing notices that port 80 stopped answering. A
readinessProbe on every app that serves HTTP would make rollouts honest:
the KS only goes Ready when the app actually answers. A livenessProbe
then covers the wedged-after-startup case.

Small job per app: an httpGet readinessProbe on the existing port, plus a
livenessProbe where the process can deadlock.

## 3. Resource requests and limits

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

## 4. Keycloak has no clients

Keycloak runs, it is built by CI with the khazadum theme, and nothing
authenticates against it. No app manifest mentions OIDC or OAuth.

Either give it clients or question why it runs at all. The easy wins are
wiki-js and Grafana: both support OIDC out of the box and would replace
their local logins. Nextcloud works too but needs the OIDC user backend
app. Each client also needs a realm, a client definition and a secret in
git, following docs/keycloak.md and docs/secrets.md.

## 5. Missing Recreate strategy

homepage, hermes and openwebui are the only single-replica apps without
`strategy: Recreate`. The other five apps have it, and so does every
deployment under infrastructure/. All three of the missing ones hold
state on hostPath volumes: openwebui has SQLite under
`/data/crypt/ai/openwebui`, hermes keeps its data there, homepage has
its config.

Without Recreate, the default RollingUpdate starts the new pod before
the old one stops. On a single node, with the same hostPath mounted in
both pods, that means two processes can share one SQLite file or config
directory at once. This is the single-writer problem from the AGENTS.md
rules, and it is a ten-minute fix per app.

## 6. The backup failure alert never resolves

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

## 7. TLS ends at the firewall

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

## 8. Small cleanups

A few things to batch whenever an app is touched anyway:

- App namespaces carry no PodSecurity labels, and most containers run as
  root. Fine for a single-operator lab, worth tightening if the cluster
  ever grows a second tenant.
- `privileged: true` on jellyfin and device-metrics-exporter is
  load-bearing for GPU device access. Leave it alone until item 9.

## 9. GPU device plugin

The proper fix for the privileged containers: an amdgpu device plugin
DaemonSet (`rocm/k8s-device-plugin`) advertising `amd.com/gpu`, so
jellyfin and device-metrics-exporter request the resource and drop
`privileged` for good. This replaces the piecemeal workarounds
documented in docs/jellyfin.md. Larger job, and partly dependent on what
upstream ships, so it sits at the bottom of the list.
