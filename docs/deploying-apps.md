# Deploying apps

Conventions that apply to every app:

- Every app owns its namespace (`apps/<app>/namespace.yaml`, deploys into it).
- Persistent data is hostPath at `/data/crypt/<app>/...`,
  `type: DirectoryOrCreate`.
- Image refs are digest-pinned (`repo:tag@sha256:<digest>`) and updated by
  Renovate ([renovate.md](renovate.md)). Resolve a digest once with
  `docker buildx imagetools inspect <ref>`.
- Single-writer / hostPort apps need `strategy: Recreate` + `replicas: 1`
  (single node: `RollingUpdate` deadlocks two pods on one host port or one
  shared SQLite/TSDB file → new pod stays `Pending`).
- Secrets go into `apps/<app>/secret.yaml`, encrypted. See
  [secrets.md](secrets.md).

## Steps

1. Create application manifests, and add them to `apps/app_name/` with a
   `kustomization.yaml`:
   ```yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   resources:
     - namespace.yaml
     - deployment.yaml
     - service.yaml
     - ingress.yaml
   ```
2. Tell flux how to manage the application with
   `clusters/khazadum/apps/app_name.yaml`:
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
3. Add the application to `clusters/khazadum/apps/kustomization.yaml` so flux
   will deploy it:
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
5. Verify deployment:
   ```sh
   # check flux status
   flux get kustomization app_name

   # check application pods
   kubectl -n namespace get pods

   # check application logs
   kubectl -n namespace logs deployment/app_name
   ```

Wiring a new app is two edits (the Flux Kustomization in
`clusters/khazadum/apps/` + the entry in that folder's `kustomization.yaml`)
plus the manifest folder. Infra apps use `clusters/khazadum/infrastructure/`
instead.
