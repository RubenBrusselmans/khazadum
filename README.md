# Khazad-dûm

>*The stone doors slowly swing open, rumbling deeply. The Fellowship enters Moria. Gandalf places a crystal onto the top of his staff; Aragorn follows last, casting a last glance at the water. Moonlight floods into the shadowy chamber.*
>
>Gimli: "Soon, Master Elf, you will enjoy the fabled hospitality of the Dwarves! Roaring fires, malt beer, ripe meat off the bone. This, my friend, is the home of my cousin, Balin."
>
>*Gandalf brings his hand around his staff, blowing upon the crystal. It glows.*
>
>Gimli: "And they call it a mine. A mine!"
>
>Boromir: "This is no mine, it's a homelab!"

## Repository

The GitHub repository is a read-only mirror. This project is self-hosted with [Forgejo](https://forgejo.org/).

## Components

- Kubernetes: [K3s](https://k3s.io/)
- Git: [Forgejo](https://forgejo.org/)
- CI: [Woodpecker](https://woodpecker-ci.org/)
- CD: [Flux CD](https://fluxcd.io)
- Secrets: [SOPS](https://getsops.io/) + [age](https://github.com/FiloSottile/age)
- Monitoring: [kube-prometheus-stack](https://github.com/prometheus-operator/kube-prometheus)
- Logging: [Loki](https://grafana.com/oss/loki/) + [Alloy](https://grafana.com/oss/alloy-opentelemetry-collector/)
- Alerting: [matrix-alertmanager-receiver](https://github.com/metio/matrix-alertmanager-receiver)
- Backups: [restic](https://restic.net/) to [Proton Drive using rclone](https://rclone.org/protondrive/)
- SSO: [Keycloak](https://www.keycloak.org/)

## Layout

```
apps/               # application manifests
clusters/khazadum/  # flux configuration
infrastructure/     # infrastructure manifests
docs/               # documentation
.woodpecker/        # pipeline
.sops.yaml          # secrets configuration
renovate.json       # automatic update config
```

## Documentation

- [architecture.md](docs/architecture.md) — network path (OPNsense → Traefik), how Flux reconciles, the two Kustomization kinds
- [deploying-apps.md](docs/deploying-apps.md) — full recipe for adding a new app (conventions, steps, verification)
- [renovate.md](docs/renovate.md) — digest pinning & automatic updates: the model, the policy, the pitfalls
- [secrets.md](docs/secrets.md) — SOPS + age: how encryption works, creating and rotating secrets
- [commands.md](docs/commands.md) — kubectl & flux command reference
- [reboot-cluster.md](docs/reboot-cluster.md) — how to reboot the cluster
- [woodpecker-ci.md](docs/woodpecker-ci.md) — CI pipelines, the keycloak image build, registry & Renovate integration
- [backup.md](docs/backup.md) — backups & snapshots: restic → rclone → Proton Drive, ZFS snapshots, restore
- [nextcloud.md](docs/nextcloud.md) — Nextcloud stack: php-overrides, DB dumps, maintenance-mode gotchas
- [keycloak.md](docs/keycloak.md) — Keycloak: CI-built themed image, update chain
- [jellyfin.md](docs/jellyfin.md) — Jellyfin: why `privileged: true` can't be dropped, GPU device access
- [roadmap.md](docs/roadmap.md) — improvement backlog from the 2026-11-20 audit: what's next for the cluster
- [AGENTS.md](AGENTS.md) — guardrails for AI-agent sessions working in this repo

## Inspiration

https://github.com/awesome-selfhosted/awesome-selfhosted
