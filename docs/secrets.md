# Secrets management

Secrets live encrypted in git. Flux decrypts at apply time.

## SOPS & age

[SOPS](https://getsops.io/) (Secrets OPerationS) is an editor of encrypted
files. Normally, if you encrypt a configuration file (like JSON or YAML), the
entire file turns into a giant block of gibberish. This causes two major
problems:

1. You can't read the file to know what settings are inside without
   decrypting it first.
2. Git sees the new block of gibberish as a completely rewritten file. You
   lose the ability to track line-by-line changes.

SOPS encrypts only the values in a file, leaving the keys in plain
text.

Example:

```yaml
database_user: admin
database_password: ENC[AES256_GCM,data:xyz123,iv:abc...]
api_key: ENC[AES256_GCM,data:qwerty,iv:def...]
```

SOPS doesn't do the encryption itself, that's done by
[age](https://github.com/FiloSottile/age) (Actually Good Encryption).
Historically, people used PGP/GPG to encrypt files. However, GPG is
notoriously complex, and is very easy to misconfigure. Age was built to
replace GPG. It is tiny, has no configuration options to mess up, and uses
modern cryptography (X25519, ChaCha20-Poly1305).

`.sops.yaml` (repo root) encrypts only `data`/`stringData` of
`apps/<app>/secret.yaml` and `infrastructure/*/secret.yaml`. Flux decrypts at
apply time via the `flux-system/sops-age` secret: an app using secrets needs
`decryption: { provider: sops, secretRef: { name: sops-age } }` in its Flux
Kustomization.

## Creating a new secret

1. Create `apps/app_name/secret.yaml` and add `- secret.yaml` to the
   application `kustomization.yaml`:
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
   ```yaml
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

## Rotating secrets

```sh
sops apps/app_name/secret.yaml
git add -A
git commit -m "modified secret for app_name"
git push
flux reconcile ks app_name --with-source
kubectl -n namespace get secret app_name
```

## Pitfall

`apiVersion: v0` in a Kustomization error means the encrypted file's
`apiVersion: v1` line got mangled. SOPS only MACs encrypted fields (a MAC,
message authentication code, is the integrity checksum it stores over the
values it encrypted), so editing that plaintext line does not invalidate
the checksum and is safe.

## Secret inventory (names only)

`nextcloud` (DB), `backup` (restic/rclone), `grafana` (admin),
`matrix-receiver` (Matrix creds), `flux-system` (git clone), `renovate`
(renovate-bot PAT), `matrix-notifier` (Flux→Matrix), `woodpecker` (Forgejo
OAuth client, agent + gRPC shared secrets). The gundabad app repo's secrets
live in that repo, not here.
