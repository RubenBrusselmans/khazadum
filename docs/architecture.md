# Architecture

How the cluster is built: the network path, the GitOps operator, and the two Kustomization kinds used in this repo.

## Network

Everything runs on a single NixOS server. Besides K3s, it also hosts a KVM
virtual machine running [OPNsense](https://opnsense.org/), which is the
firewall (interfaces are passed to the VM as macvtap devices).

OPNsense provides:

- Firewall
- Wireguard VPN server
- DNS & DHCP server
- Reverse proxy ([Caddy](https://caddyserver.com/))
- ACME client
- IDS/IPS ([Suricata](https://suricata.io/))

Caddy terminates the public TLS (Let's Encrypt, via its ACME client) and then
re-encrypts: it forwards to Traefik over HTTPS, so traffic between OPNsense
and the cluster is always encrypted.

## GitOps & Flux

Git is the single source of truth. Flux runs as a set of controllers inside
the cluster (in the flux-system namespace) and continuously watches this
repository.

It operates on a reconciliation loop: on a set interval, Flux pulls the latest
state and converges the cluster to match. Think of it as an automated
`git pull` → `kubectl apply` running endlessly in the background (just like
systemd keeps a service aligned with its unit file).

- `source-controller` — fetches the repo and keeps a local snapshot (artifact) from Git or Helm
- `kustomize-controller` — builds & applies raw Kubernetes manifests (YAML)
- `helm-controller` — manages Helm chart releases (`helm install/upgrade`)
- `notification-controller` — routes events & alerts to Matrix, webhooks, etc.

Failures alert a Matrix room (`infrastructure/notifications/`): a broken
rollout flips its Kustomization not-Ready and the alert arrives within
minutes.

## Kustomization

When deploying with Flux, you will encounter two different types of
kustomization files. Each app requires both:

- Kubernetes Kustomize (`kustomize.config.k8s.io/v1beta1`, lives in the
  app manifest folder `apps/app_name/`): the builder. It assembles the final
  deployment: it combines raw manifests, applies patches (e.g., staging vs.
  production), and filters which files to include. It defines *what* to
  deploy.
- Flux Kustomization (`kustomize.toolkit.fluxcd.io/v1`, lives in the
  Flux config folder `clusters/khazadum/apps/`): the delivery CRD. It tells
  the Flux controller which path in Git to sync, how often to check for
  updates, and how to handle configuration drift. It defines *how* and
  *when* to deploy.
