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

[Khazad-dûm](https://lotr.fandom.com/wiki/Khazad-d%C3%BBm), also known as the Dwarrowdelf, the Mines of Moria, or simply Moria, was an underground kingdom beneath the Misty Mountains. It was known for being the ancient realm of the Dwarves of Durin's Folk, and the most famed of all Dwarven realms.

It is also my homelab.

## Repository

The GitHub repository is a read-only mirror, I self-host this project with [Forgejo](https://forgejo.org/). Please don't open issues or PRs on GitHub.

## Components

I prefer simplicity and low resource usage over fancy features:

- Kubernetes distribution: [K3s](https://k3s.io/)
- GitOps operator: [Flux CD](https://fluxcd.io)
- Source control: [Forgejo](https://forgejo.org/)
- Secrets management: [SOPS](https://getsops.io/) + [age](https://github.com/FiloSottile/age)
- Monitoring: [kube-prometheus-stack](https://github.com/prometheus-operator/kube-prometheus)
- Logging: [Loki](https://grafana.com/oss/loki/) + [Alloy](https://grafana.com/oss/alloy-opentelemetry-collector/)
- Alerting: [matrix-alertmanager-receiver](matrix-alertmanager-receiver)
- Backups: [restic](https://restic.net/) to [Proton Drive using rclone](https://rclone.org/protondrive/)

## Layout

```
apps/               # application manifests
clusters/khazadum/  # flux configuration
infrastructure/     # infrastructure manifests
.sops.yaml          # secrets configuration
README.md           # one README to rule them all
renovate.json       # automatic update config
```

## Network

Everything runs on a single NixOS server. Besides K3s, it also hosts a KVM virtual machine running [OPNsense](https://opnsense.org/), which serves as my firewall (interfaces are passed to the VM as macvtap devices).

OPNsense acts as my:
- Firewall
- Wireguard VPN server
- DNS & DHCP server
- Reverse proxy ([Caddy](https://caddyserver.com/))
- ACME client
- IDS/IPS ([Suricata](https://suricata.io/))

Right now, Caddy terminates TLS and forwards the request over plain HTTP to K3s (with Traefik as ingress controller). So encryption currently ends at the firewall. Traffic between OPNsense and the cluster itself is plain HTTP. This traffic never leaves the node, but I plan on enabling TLS on Traefik as well, and re-encrypt traffic between the firewall and the cluster.

## GitOps & Flux

Git is the single source of truth. Flux runs as a set of controllers inside the cluster (in the flux-system namespace) and continuously watches this repository.

It operates on a reconciliation loop: on a set interval, Flux pulls the latest state and converges the cluster to match. Think of it as an automated git pull → kubectl apply running endlessly in the background (just like systemd keeps a service aligned with its unit file).

| controller | Role |
| --- | --- |
| source-controller | Fetches the repo and keeps a local snapshot (artifact) from Git or Helm |
| kustomize-controller | Builds & applies raw Kubernetes manifests (YAML) |
| helm-controller | Manages Helm chart releases (`helm install/upgrade`) |
| notification-controller | Routes events & alerts to Matrix, webhooks, etc. |

## Kustomization

When deploying with Flux, you will encounter two different types of kustomization files. Each app requires both:

| | **Kubernetes Kustomize** | **Flux Kustomization** |
| --- | --- | --- |
| **API Version** | `kustomize.config.k8s.io/v1beta1` | `kustomize.toolkit.fluxcd.io/v1` |
| **Location** | App manifest folder `apps/app_name/` | Flux config folder `clusters/khazadum/apps/` |
| **Role** | **Builder:** Assembles the final deployment. It combines raw manifests, applies patches (e.g., staging vs. production), and filters which files to include.  | **Delivery:** A CRD that orchestrates delivery. It tells the Flux controller which path in Git to sync, how often to check for updates, and how to handle configuration drift. |
| **Summary** | Defines *what* to deploy. | Defines *how* and *when* to deploy. |

## Application deployment

1. Create application manifests, and add them to `apps/app_name/` with a `kustomization.yaml`:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - namespace.yaml
     - deployment.yaml
     - service.yaml
     - ingress.yaml
   ```
2. Tell flux how to manage the application with `clusters/khazadum/apps/app_name.yaml`:
   ```yaml
   apiVersion: kustomize.toolkit.fluxcd.io/v1
   kind: Kustomization
   metadata:
     name: app_name
     namespace: flux-system
   spec:
     interval: 10m0s
     path: ./apps/app_name
     sourceRef: { kind: GitRepository, name: flux-system }
     prune: true                       # delete cluster objects removed from git
     force: true                       # take over hand-applied objects
     # decryption:                     # if SOPS secrets are used (secret.yaml)
     #   provider: sops
     #   secretRef: { name: sops-age }
   ```
3. Add the application to `clusters/khazadum/apps/kustomization.yaml` so flux will deploy it:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - existing_app.yaml
     - app_name.yaml  # new app added here
   ```
4. Commit, push, reconcile:
   ```sh
   git add -A
   git commit -m "deploy app_name"
   git push
   # force reconciliation (optional, happens automatically)
   flux reconcile kustomization flux-system --with-source   # update the operator first
   flux reconcile kustomization app_name --with-source      # next apply app_name
   ```
5. Verify deployment
   ```sh
   # check flux status
   flux get kustomization app_name

   # check application pods
   kubectl -n namespace get pods

   # check application logs
   kubectl -n namespace logs deployment/app_name
   ```

## Renovate

Flux reads YAML files from Git and applies them to Kubernetes. Renovate updates those YAML files when a new version is released. It is a bot that scans your Git repository, finds container images and Helm charts, and automatically opens pull requests to update them.

My goal was to have both automatic updates AND digest pinning. Renovate makes this possible.

- With Docker or Podman, you pull an image by a tag, like nginx:1.24.0 or nextcloud:latest
- But tags are "mutable" (they can change): a new version could be deployed under the same tag
- So even with the same tag, a cluster could be running different versions of software across multiple pods
- This can be fixed with digest pinning, by using the sha256 hash of the exact build
- PROBLEM: digests don't get security updates
--> SOLUTION: Renovate

Example:
1. START: git repository says `image: loki@sha256:abc....`
2. The maintainer fixes a bug and pushes a new build. The latest tag now points to `sha256:def....`
3. Renovate checks the registry and sees the version we want has a new digest
4. Renovate changes the config to `image: loki@sha256:def....` and creates a PR
5. Depending on your configuration, Renovate might even auto-merge the PR (minor change or bug fix). Or you accept the PR manually.
6. Flux sees the merge (new git commit), and reconciles the cluster to the new state (rolls out the new image).

I prefer OCI Helm registries over HTTP because of digest pinning.

With OCI, you pin the entire chart artifact with a single digest. With HTTP registries, there is no chart-wide digest. To achieve the same security, you would have to manually extract and pin the digest of every single container image defined inside that chart's values.yaml, which is messy and unmaintainable. 

For raw manifests, Renovate handles image digest pinning automatically. 

**Alloy lacks an official OCI registry, so it is an exception (using a HTTP registry). It receives automated updates but has no digest pinning.**

### Why Renovate instead of only Flux?

Flux also has image automation, but it cannot update Helm chart versions or calendar-versioned tags like `2026.8.5` (used by matrix-receiver). It also doesn't have easily reviewable PRs.

## Secrets management

Secrets live encrypted in git, Flux decrypts at apply time.

### SOPS & age

SOPS (Secrets OPerationS) is an **editor** of encrypted files. Normally, if you encrypt a configuration file (like JSON or YAML), the entire file turns into a giant block of gibberish. This causes two major problems:

1. You can't read the file to know what settings are inside without decrypting it first.
2. Git sees the new block of gibberish as a completely rewritten file. You lose the ability to track line-by-line changes.

SOPS encrypts **only the values** in a file, leaving the **keys** in plain text. 

Example:

```yaml
database_user: admin
database_password: ENC[AES256_GCM,data:xyz123,iv:abc...]
api_key: ENC[AES256_GCM,data:qwerty,iv:def...]
```

SOPS doesn't do the encryption itself, that's done by **age** (Actually Good Encryption). Historically, people used PGP/GPG to encrypt files. However, GPG is notoriously complex, and is very easy to misconfigure. Age was built to replace GPG. It is tiny, has no configuration options to mess up, and uses modern cryptography (X25519, ChaCha20-Poly1305). 

### Creating a new secret

1. Create `apps/app_name/secret.yaml` and add `- secret.yaml` to the application `kustomization.yaml`:
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: app_name
     namespace: namespace
   type: Opaque
   stringData:
     SOME_KEY: "secret123"
   ```
2. Reference it from `deployment.yaml`:
   ```yaml
   env
     - name: SOME_KEY
       valueFrom:
         secretKeyRef:
           name: app_name
           key: SOME_KEY
   ```
3. Tell Flux to decrypt secrets in the flux kustomization `app_name.yaml`:
   ```
   spec:
     decryption:
       provider: sops
       secretRef:
         name: sops-age
   ```
4. Encrypt, commit, reconcile, verify:
   ```sh
   sops --encrypt --in-place apps/app_name/secret.yaml
   git add -A
   git commit -m "added secret for app_name"
   git push
   flux reconcile ks app_name --with-source
   kubectl -n namespace get secret app_name
   ```

### Rotating secrets

```sh
sops apps/app_name/secret.yaml
git add -A
git commit -m "modified secret for app_name"
git push
flux reconcile ks app_name --with-source
kubectl -n namespace get secret app_name
```

## Command reference

### kubectl

```sh
# common kubectl commands
kubectl cluster-info                       # cluster information
kubectl get nodes                          # list nodes
kubectl get namespaces                     # list namespaces
kubectl get pods -n namespace              # list pod in namespace
kubectl get pods -A                        # list pods in all namespaces
kubectl describe node node_name            # node information
kubectl describe pod pod_name -n namespace # pod information
kubectl apply -f file_or_folder            # deploy pods
kubectl delete pods pod_name -n namespace  # delete pods
kubectl delete pods --all -n namespace     # delete all pods
kubectl logs pod_name -n namespace         # show logs of pod
kubectl logs pod_name -n namespace -f      # follow logs of pod
kubectl exec -it pod_name -- /bin/sh       # exec command in pod

# list containers inside pod
kubectl get pods -n observability -o jsonpath='{.spec.containers[*].name}' \
 kube-prometheus-stack-grafana-5db86f7cc-hqr65

# list containers inside pod using deployment name
kubectl get -n observability -o jsonpath='{.spec.template.spec.containers[*].name}' \
 deployments/kube-prometheus-stack-grafana

# enter one of the containers in the pod
kubectl exec -it -n observability deployments/kube-prometheus-stack-grafana -c grafana -- /bin/sh
```

### flux

```sh
# common flux commands
flux get all -A                    # list all resources & status
flux get kustomizations -A         # list all kustomizations & status
flux get helmreleases -A           # list all Helm releases & status
flux get sources all -A            # list all sources & status
flux get sources git -A            # list git sources & status
flux get sources helm -A           # list helm sources & status
flux check                         # controllers & CRDs healthy?
flux suspend kustomization app     # pause reconciliation (troubleshooting)
flux resume kustomization app      # resume reconciliation

# trigger a reconciliation of sources and resources
flux reconcile kustomization flux-system --with-source  # update the parent / operator first
flux reconcile kustomization app_name --with-source     # next update / build the application

# show all flux objects that are not ready
flux get all -A --status-selector ready=false

# looking for controller errors
flux logs --all-namespaces --level=error

# show flux warning events
kubectl get events -n flux-system --field-selector type=Warning

# list all sources & status with kubectl
kubectl get gitrepositories.source.toolkit.fluxcd.io -A
kubectl get helmrepositories.source.toolkit.fluxcd.io -A

# list all kustomizations or Helm releases & status with kubectl
kubectl get kustomizations.kustomize.toolkit.fluxcd.io -A
kubectl get helmreleases.helm.toolkit.fluxcd.io -A
kubectl get helmcharts.source.toolkit.fluxcd.io -A
```

## reboot cluster

```sh
sudo systemctl stop k3s
sudo reboot

sudo zfs load-key data/crypt
sudo zfs mount data/crypt
mountpoint /data/crypt
sudo systemctl start k3s
```

## Inspiration

https://github.com/awesome-selfhosted/awesome-selfhosted