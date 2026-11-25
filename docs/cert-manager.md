# Certificates (cert-manager)

cert-manager runs a cluster-internal CA and signs the wildcard certificate
Traefik serves. Public TLS stays at Caddy (Let's Encrypt). The root is named
`k3s khazadum` because it covers this cluster only; other khazadum.net
services outside the cluster are not signed by it.

> Client --https--> Caddy (OPNsense) --https--> Traefik (K3s) --http--> pods

## Objects

Controller in `infrastructure/cert-manager/`, issuers and Traefik TLS in
`infrastructure/pki/` — a second Flux Kustomization that depends on
cert-manager being Ready, because the pki objects need the CRDs from the
cert-manager chart to exist before they can apply.

- ClusterIssuers `k3s-khazadum-root` (selfSigned) and `k3s-khazadum-ca` (ca).
- Root CA Certificate `k3s-khazadum-root-ca`: 10y, RSA-4096,
  `rotationPolicy: Never`.
- Certificate `wildcard-khazadum-net` in kube-system: `khazadum.net` +
  `*.khazadum.net`, 90d, ECDSA.
- TLSStore `default` in kube-system: Traefik serves that secret as its
  default certificate, so no Ingress needs a `spec.tls` block.

## Renewal

Leaf: 90d, renewed at 60d, automatic; Traefik picks up secret changes
without a restart. Root: renewed in place at ~2/3 of its lifetime with the
same key (`rotationPolicy: Never`), so the OPNsense trust anchor stays
valid. Re-import the root only after a deliberate key regeneration.

A cluster rebuild regenerates the root (it is not in git): export and
re-import into OPNsense once afterwards.

## OPNsense (one-time)

```sh
kubectl -n cert-manager get secret k3s-khazadum-root-ca \
  -o go-template='{{index .data "ca.crt"}}' | base64 -d > k3s-khazadum-root-ca.crt
```

Import the file (System → Trust → Authorities), then set Caddy's upstreams
to `https://tower.khazadum.net` with that authority selected for
verification.

## Verify

```sh
kubectl -n cert-manager get clusterissuers
kubectl -n kube-system get certificate
kubectl -n kube-system get secret wildcard-khazadum-net
openssl s_client -connect 192.168.10.2:443 -servername auth.khazadum.net </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer
```
