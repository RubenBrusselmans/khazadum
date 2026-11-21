# Renovate & image pinning

Flux reads YAML files from Git and applies them to Kubernetes. Renovate
updates those YAML files when a new version is released. It is a bot that
scans this Git repository, finds container images and Helm charts, and
automatically opens pull requests to update them.

The goal is to have both automatic updates and digest pinning. Renovate makes
this possible.

## Why digest pinning

- With Docker or Podman, you pull an image by a tag, like `nginx:1.24.0` or
  `nextcloud:latest`.
- But tags are "mutable" (they can change): a new version could be deployed
  under the same tag.
- So even with the same tag, a cluster could be running different versions of
  software across multiple pods.
- This can be fixed with digest pinning, by using the sha256 hash of the
  exact build.
- Catch: a pinned digest never changes by itself, so it also never picks up
  security updates.
- Fix: Renovate, which watches the registry and opens the PRs that move the
  digest.

Example update cycle:

1. The repo pins `image: loki@sha256:abc....`
2. The maintainer fixes a bug and pushes a new build. The latest tag now
   points to `sha256:def....`
3. Renovate checks the registry and sees that the pinned version has a new digest
4. Renovate changes the config to `image: loki@sha256:def....` and creates a PR
5. Depending on your configuration, Renovate might even auto-merge the PR
   (minor change or bug fix). Or you accept the PR manually.
6. Flux sees the merge (new git commit), and reconciles the cluster to the
   new state (rolls out the new image).

## The pinning model

**The rule: digest-pin exactly what automation manages atomically. Nothing
else, and never by hand.**

- Raw manifests: `image: repo:tag@sha256:<digest>`. Renovate rewrites the
  full ref in one atomic PR, and the digest is also the *update signal* for
  `latest`-riding images (jellyfin, …), which would otherwise float
  unmanaged. This buys three things: the cluster only ever runs builds whose
  origin you can trace, the same digest always means the same bytes, and a
  revert is a single commit that points the digest back at the previous
  build.
- CI-built images: Woodpecker builds push `latest` + `sha-<sha>` to the
  Forgejo registry. Deployments pin `latest@sha256:<digest>`. Builds run with
  kaniko `--reproducible`, so unchanged inputs keep the digest: Renovate
  sees nothing to bump, and the digest moves only when the image content
  moves. Current: keycloak (see [keycloak.md](keycloak.md) and
  [woodpecker-ci.md](woodpecker-ci.md)).
- Helm charts: `OCIRepository` with `ref.tag` + `ref.digest` (ghcr).
  Renovate treats it as *one* dependency and moves both fields in one PR.
  Images ride the chart's own defaults, transitively covered by the chart's
  content hash. Current: kube-prometheus-stack, loki. alloy has no upstream
  OCI mirror: exact `chart.spec.version` on the HTTP repo for now.
- **Never add `image:` overrides to HelmRelease values.** Chart values split
  images into paired `tag:` + `sha:`/`digest:` fields in per-chart formats.
  Renovate's replace-verification demands `sha256:`-prefixed digests in the
  file while the prometheus/grafana chart templates require bare hex (they
  prepend the prefix at render). The two formats can't be reconciled in one
  atomic edit, so a half-updated pair stays broken until it is hand-fixed.
  Manual pairing was tried and missed twice within 24h: **manual digest
  labor is banned, full stop.**

OCI Helm registries are preferred over HTTP because of digest pinning. With OCI,
you pin the entire chart artifact with a single digest. With HTTP registries,
there is no chart-wide digest. To achieve the same security, you would have
to manually extract and pin the digest of every single container image
defined inside that chart's values.yaml, which is messy and unmaintainable.

## Renovate is the only versioning authority

Self-hosted CronJob every 2h (`infrastructure/renovate/`), dedicated
`renovate-bot` Forgejo account, policy in `renovate.json`:

- digest + patch + minor PRs automerge
- major versions are human PRs
- matrix-receiver (it delivers the alert pages) is always reviewed PRs
- mariadb constrained to `^11` (nextcloud support matrix)
- CI status does not gate automerge (`ignoreTests: true`). Flipping
  that is a deliberate change: automerge would stall whenever CI is red.

### Why Renovate instead of only Flux?

Flux also has image automation (ran here 2026-08-17, decommissioned), but:

- Flux's image-automation setters write one complete ref into one field:
  no chart versions, no split values fields, no calendar versioning
  (matrix-receiver `2026.8.5` string-sorts October before August)
- Flux's native semver-range alternative upgrades in-cluster with no git
  trail: the cluster changes but the repo doesn't, so there is nothing to
  revert and the repo can't answer "what's running?"
- changes don't arrive as reviewable PRs

Renovate covers everything through reviewable PRs.

## Operational notes

- Renovate races your pushes: it automerges to `main` continuously, so
  pushes will sometimes be rejected. `git pull --rebase` is routine.
- Cadence `schedule: before 6am`: PR/branch creation only happens in that
  window. "No PR appeared" during the day is expected, not broken.
- Manual run:
  `kubectl -n renovate create job renovate-manual --from=cronjob/renovate`.
