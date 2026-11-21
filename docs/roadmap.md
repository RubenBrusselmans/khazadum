# Improvement roadmap

Findings from a full audit of the manifests, done 2026-11-20, ranked by how
much cluster improvement they buy per hour of work. Items are removed once
they are done, so everything here is still open. New findings get added at
the bottom; the numbering only changes when an item is removed.

## 1. TLS ends at the firewall

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

## 2. Small cleanups

A few things to batch whenever an app is touched anyway:

- App namespaces carry no PodSecurity labels, and most containers run as
  root. Fine for a single-operator lab, worth tightening if the cluster
  ever grows a second tenant.
- `privileged: true` on jellyfin and device-metrics-exporter is
  load-bearing for GPU device access. Leave it alone until item 3.

## 3. GPU device plugin

The proper fix for the privileged containers: an amdgpu device plugin
DaemonSet (`rocm/k8s-device-plugin`) advertising `amd.com/gpu`, so
jellyfin and device-metrics-exporter request the resource and drop
`privileged` for good. This replaces the piecemeal workarounds
documented in docs/jellyfin.md. Larger job, and partly dependent on what
upstream ships, so it sits at the bottom of the list.
