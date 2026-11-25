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
- Certificates: [cert-manager](https://cert-manager.io/)
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

## Inspiration

https://github.com/awesome-selfhosted/awesome-selfhosted
