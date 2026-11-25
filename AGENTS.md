# AGENTS.md

Guardrails for AI-agent sessions working in this repo. `README.md` is for
human readers, `docs/` holds the documentation.

## What this repo is

- Declarative desired state of the single-node `khazadum.net` k3s cluster,
  reconciled by Flux. Almost pure Kubernetes YAML, no build or lint tooling
  locally. Editing manifests in git *is* the change; Flux applies it.
- CI is Woodpecker (kubernetes backend, Forgejo forge); its first pipeline
  is the YAML parse-check.
- One external app repo (`gundabad`) is pulled in through
  `clusters/khazadum/apps/gundabad.yaml`. Its manifests and secrets live
  there, not here.

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
- [cert-manager.md](docs/cert-manager.md) — internal PKI: k3s-only root CA, Traefik wildcard cert, OPNsense trust import
- [roadmap.md](docs/roadmap.md) — improvement backlog from the 2026-11-20 audit: what's next for the cluster

## Hard safety rules

1. Never `git commit`, never `git push`. All commits and pushes are
   operator-run; the agent edits, validates, and stages when asked. After
   every change, propose a Conventional Commits message
   (`type(scope): subject`) for the operator to commit verbatim.
2. Never `kubectl apply` an app Flux owns. Edit git, commit, push,
   `flux reconcile ks <app> --with-source`. Hand-applying causes
   field-manager conflicts. (`clusters/khazadum/flux-system/gotk-*.yaml`
   are flux-generated, do-not-edit files.)
3. Never delete a Flux `Kustomization` CR to "roll back"; `prune: true`
   deletes the whole app. To pause: `flux suspend ks <app>`. A suspended
   KS never runs its deletion finalizer, so removing it from git while
   suspended leaves all its resources behind. Always
   `flux resume ks <app>` before removing it from git.
4. Images and charts follow the pinning model
   ([docs/renovate.md](docs/renovate.md)). Never hand-edit a digest or
   version to "update" anything; adjust `renovate.json` policy instead.
5. Single-writer / hostPort apps need `strategy: Recreate` +
   `replicas: 1`. Single node: `RollingUpdate` deadlocks two pods on one
   host port or one shared SQLite/TSDB file, and the new pod stays
   `Pending`.
6. Helm is allowed anywhere (per-app call), but must satisfy the pinning
   rules: prefer OCI charts, plain HTTP charts only when no OCI mirror
   exists.

## Verify (no test suite)

Edit git → push → Flux reconcile → read-only cluster checks.

Local YAML parse-check (CI runs it on every push; local run useful when CI
is broken or before pipeline changes):

```sh
nix-shell -p python3Packages.pyyaml --run 'python3 - <<EOF
import yaml, glob, sys
bad = 0
for f in sorted(sum((glob.glob(p + "/**/*.yaml", recursive=True)
                     for p in ("apps", "infrastructure", "clusters", ".woodpecker")), [])):
    try:
        list(yaml.safe_load_all(open(f)))
    except yaml.YAMLError as e:
        bad += 1
        print(f"FAIL {f}: {e}")
if bad:
    sys.exit(1)
print("all YAML parse OK")
EOF'
```

(Syntax only; schema errors still need review.)

Cluster checks:

```sh
flux get ks                                   # all Kustomizations Ready?
flux get sources git                          # git source cloned OK?
flux reconcile ks <app> --with-source         # apply a single app now
flux reconcile ks flux-system --with-source   # parent: picks up new KS files
kubectl -n <ns> get pods                      # 1/1 Running
```

Ownership check (Flux vs hand-applied):

```sh
kubectl -n <ns> get deploy <name> -o jsonpath='{.metadata.managedFields[*].manager'
# kustomize-controller = Flux owns it; kubectl-client-side-apply = not yet adopted
```

More commands: [docs/commands.md](docs/commands.md).

## Pinning (rules only; full model in [docs/renovate.md](docs/renovate.md))

- Digest-pin exactly what automation manages atomically. Nothing else,
  never by hand.
- Raw manifests: `image: repo:tag@sha256:<digest>` (the digest is also the
  update signal for `latest`-riding images).
- CI-built images: pin `latest@sha256:<digest>` (kaniko `--reproducible`
  keeps digests stable across no-op rebuilds).
- Helm charts: `OCIRepository` with `ref.tag` + `ref.digest` as one
  dependency; HTTP charts pin an exact `chart.spec.version` (alloy).
- Never add `image:` overrides to HelmRelease values. Manual digest labor
  is banned.
- Renovate is the only versioning authority (policy: `renovate.json`).

## Secrets

- Secret values and the age key are operator-confidential. The agent never
  sees them. SOPS + age at rest, Flux decrypts at apply time.
- Agent workflow: write `"<operator fills>"` placeholders in
  `apps/<app>/secret.yaml`, wire `secretKeyRef`, list the file in the app
  kustomization (commented out until encrypted), and add
  `decryption: { provider: sops, secretRef: { name: sops-age } }` to the
  Flux Kustomization.
- Never run `sops --encrypt` / `--decrypt`, never edit real values, never
  decode secrets. All sops operations are operator-run.
- Verify reconciliation by existence + DATA-key count only:

```sh
kubectl -n <ns> get secret <app>    # DATA column = number of keys; that's enough
```

Details: [docs/secrets.md](docs/secrets.md).

## Operational notes

- Renovate automerges to `main` continuously and races your pushes;
  `git pull --rebase` when a push is rejected.
- Renovate creates PRs only before 6am; daytime silence is expected.
  Manual run:
  `kubectl -n renovate create job renovate-manual --from=cronjob/renovate`.
- A broken rollout flips its KS not-Ready and alerts Matrix within minutes
  (`infrastructure/notifications/`). Verify after every push.
- `privileged: true` on jellyfin and device-metrics-exporter is required
  (removing it breaks GPU device access). Do not fix it piecemeal.
  See [docs/jellyfin.md](docs/jellyfin.md).
- The improvement backlog lives in [docs/roadmap.md](docs/roadmap.md)
  (audit 2026-11-20). Check it before proposing new work; remove entries
  when they are done.

## Conventions

- Two `Kustomization` kinds: the kustomize bundler
  (`kustomize.config.k8s.io`, in `apps/<app>/kustomization.yaml`) and the
  Flux KS (`kustomize.toolkit.fluxcd.io`, in `clusters/khazadum/apps/`).
  "KS" always means the Flux one.
- Every app owns its namespace (`apps/<app>/namespace.yaml`).
- New app = two edits: `clusters/khazadum/apps/<app>.yaml` (with
  `wait: true`) and an entry in `clusters/khazadum/apps/kustomization.yaml`,
  plus the manifest folder. Infra apps use
  `clusters/khazadum/infrastructure/`. Recipe:
  [docs/deploying-apps.md](docs/deploying-apps.md).
- New image: `image: repo:tag@sha256:digest`; resolve the digest once with
  `docker buildx imagetools inspect <ref>`. Renovate picks it up
  automatically.
- Every app and infra component ships a default-deny ingress
  NetworkPolicy with a traefik allow rule, like
  `apps/*/networkpolicy.yaml`.
- Persistent data is hostPath at `/data/crypt/<app>/...`,
  `type: DirectoryOrCreate`.
- Forgejo (app `git`) is Flux's own git source: if git is down, Flux can't
  see new commits but running apps keep working. Push to
  `https://git.khazadum.net/ruben/khazadum.git`.
