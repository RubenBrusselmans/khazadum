# Woodpecker CI

CI for the cluster: Woodpecker server + agent, kubernetes backend (pipeline
steps run as pods), Forgejo as the forge. Web UI at
https://woodpecker.khazadum.net.

Currently runs two pipelines on `ruben/khazadum`: a YAML parse-check and the
keycloak image build. See `.woodpecker/*.yaml`.

## How a pipeline runs

A push to Forgejo starts this chain:

1. Forgejo notifies `woodpecker-server` with a webhook (HTTP, port 8000).
2. The server queues the pipeline and hands it to `woodpecker-agent` over
   gRPC (port 9000).
3. The agent runs the pipeline on the kubernetes backend: every step
   becomes a pod (`wp-*`) in the woodpecker namespace, and a `plugin-git`
   step clones the repo before anything else runs.
4. Steps run in order, one pod each, and a finished pod is deleted. The
   workspace moves between steps through a temporary PVC per pipeline (a
   persistent volume claim from the `local-path` storage class,
   ReadWriteOnce).
5. Logs, pipeline status and the manual trigger UI live in the web UI.

## Pipelines

### `.woodpecker/validate.yaml`

- Triggers: every push (all branches, including Renovate PR branches) + manual.
- Alpine + py3-yaml. Runs the same parse-check as AGENTS.md. Globs include
  `.woodpecker` itself (the CI file parse-checks itself).
- A cheap smoke test for the whole chain: webhook, clone, kubernetes
  backend.

### `.woodpecker/build-keycloak-image.yaml`

- Triggers: push to `main` touching `apps/keycloak/image/**` + manual.
  (Renovate's FROM-bump merges land this way.)
- Builds `apps/keycloak/image/Containerfile` with kaniko
  (`executor:v1.24.0-debug`) and pushes to the Forgejo registry:
  - `git.git.svc.cluster.local:3000/ruben/keycloak-khazadum:latest`
  - `:sha-<commit-sha-7>`: registry auth via repo secrets
    `registry_user`/`registry_token` written to `config.json` in the step.
- Memory limit 2Gi (`backend_options`) so `kc.sh build` doesn't starve the
  node's other tenants.
- `--reproducible`: identical inputs must produce a byte-identical image,
  otherwise every rebuild changes the `latest` digest and Renovate churns
  out pointless keycloak rollouts.

## How Renovate fits in

Renovate (self-hosted, see `infrastructure/renovate/`) updates every image
ref in this repo. The managers that matter here:

- woodpecker manager → image refs in `.woodpecker/*.yaml`
- dockerfile manager → `apps/keycloak/image/Containerfile` FROM digests
- kubernetes manager → server/agent/deployment image refs

The keycloak loop runs end to end: the dockerfile manager opens a PR for a
new `FROM` digest, the merge triggers the build pipeline, the build pushes
a new image digest, and the kubernetes manager opens the PR that moves the
`deployment.yaml` pin to it. No-op rebuilds produce no new digest (the
`--reproducible` flag, see the build pipeline above), so Renovate stays
quiet unless the image content changed.

The pinning rules live in [renovate.md](renovate.md), the keycloak-specific
update chain in [keycloak.md](keycloak.md). Nothing in this loop is
hand-edited.
