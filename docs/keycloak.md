# Keycloak

Single-sign-on for the cluster. External URL: auth.khazadum.net
(ingress.yaml). Single-pod stack: postgres sidecar + keycloak.

The keycloak image is built in this repo, not pulled from upstream: the
Containerfile bakes in the login themes (`image/themes/`) and a `kc.sh build`
optimization pass. Pipeline: `.woodpecker/build-keycloak-image.yaml`
(full story: [woodpecker-ci.md](woodpecker-ci.md)).

## Rules

- The image ref is `latest@sha256:<digest>` and must stay that way. The
  whole update chain depends on it:
  Renovate bumps the Containerfile `FROM` digest → Woodpecker rebuilds →
  new image digest → Renovate digest-bump PR on `deployment.yaml` →
  automerge → Flux rolls. Nothing is hand-edited. Kaniko `--reproducible`
  keeps no-op rebuilds digest-identical so Renovate has nothing to bump.
- `strategy: Recreate` + `replicas: 1`: postgres data is on hostPath.
- Postgres is digest-pinned in the same file. Renovate updates it like any
  raw-manifest image.

## Image build facts

- Build context: `image/` (Containerfile + themes). A push touching
  `image/**` on main triggers the build (this is how Renovate's FROM-bump merges
  trigger builds). Manual runs from the Woodpecker UI work too.
- Registry: Forgejo package `ruben/keycloak-khazadum`, public, anonymous
  pull works. Tags pushed: `latest` + `sha-<commit-sha-7>`.
- Kaniko needs the `-debug` executor image (shell-less otherwise).
- `KC_*` env (hostname, proxy headers, health) is baked in the Containerfile
  build stage, not set in the Deployment.
- The build is memory-limited to 2Gi so `kc.sh build` doesn't starve the
  node's other tenants.

## Themes

Custom login theme in `image/themes/khazadum/`. Theme changes ship through
the same pipeline: edit themes → commit → build runs → digest bump PR →
rollout.

## Clients

- wiki.js (bypass: https://wiki.khazadum.net/login?all=1)
- grafana (bypass: https://grafana.khazadum.net/login?disableAutoLogin)
- forgejo (bypass: https://git.khazadum.net/user/login)
- nextcloud (bypass: https://cloud.khazadum.net/login?direct=1)