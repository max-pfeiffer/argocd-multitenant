# step-ca CA material — procedure, not material

This file documents how the CA secrets are produced. **It never contains key material** (guardrail 12).

## Why this is not automated

The `step-certificates` chart generates a fresh root CA when it does not find existing secrets. On a re-sync, a
re-install, or a namespace recreation that silently rotates the cluster's root of trust and invalidates every
certificate issued from it (guardrail 30). So the chart is configured to generate nothing, and the material is
committed here in sealed form.

Turning generation off is three mutually-exclusive blocks in `infra/apps/step-certificates.yaml`, not one:

```yaml
existingSecrets: {enabled: true, ca: true, certsAsSecret: true, configAsSecret: true}
bootstrap:       {enabled: false, secrets: false, configmaps: false}
inject:          {enabled: false}
```

**`existingSecrets.*` are boolean flags, not secret names.** Chart 1.30.1 derives every name from the Helm release
(`fullname` = `step-certificates`), so the names below are fixed. Passing a name instead of `true` is truthy — the
chart accepts it and mounts a different secret than the one you wrote, which fails at pod start rather than at sync.

| Secret | Mounted at | Keys |
|---|---|---|
| `step-certificates-certs` | `/home/step/certs` | `root_ca.crt`, `intermediate_ca.crt` |
| `step-certificates-config` | `/home/step/config` | `ca.json`, `defaults.json` |
| `step-certificates-secrets` | `/home/step/secrets` | `root_ca_key`, `intermediate_ca_key` |
| `step-certificates-ca-password` | `/home/step/secrets/passwords/password` | `password` |
| `step-certificates-provisioner-password` | not mounted — read by step-issuer | `password` |

## Prerequisites

- The sealed-secrets controller must be running (wave −2) before anything can be sealed: `kubeseal` fetches the
  sealing certificate from it.
- **The sealing key must already be backed up** (guardrail 31). Sealing before backing up the key means the material
  in Git is unrecoverable if the controller's key is lost.

## One-time bootstrap

Run once, on a trusted machine, before `infra/apps/step-certificates.yaml` first syncs. `step ca init` produces
exactly the layout the chart mounts — this is the same work the chart's bootstrap Job would do, done where the output
can be sealed and reviewed instead of landing straight in the cluster.

```bash
export STEPPATH=$(mktemp -d)

# 1. Two passwords. They are never reused and never leave this machine in plaintext.
LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom | head -c 40 > ca_password.txt
LC_ALL=C tr -dc 'A-Za-z0-9' </dev/urandom | head -c 40 > provisioner_password.txt

# 2. Create the PKI: root, intermediate, ca.json with a JWK provisioner, defaults.json.
#    --dns must cover how cert-manager reaches the CA in-cluster; --with-ca-url must match
#    spec.url in step-clusterissuer.yaml exactly.
step ca init \
  --name "Platform Root CA" \
  --dns "step-certificates.step-ca.svc.cluster.local,step-certificates,127.0.0.1" \
  --address ":9000" \
  --provisioner "admin@amt.lan" \
  --with-ca-url "https://step-certificates.step-ca.svc.cluster.local" \
  --password-file ca_password.txt \
  --provisioner-password-file provisioner_password.txt

# 3. `step ca init` writes absolute paths pointing at $STEPPATH. The container sees /home/step,
#    so rewrite them or step-ca starts up unable to find its own root certificate.
sed -i '' "s|$STEPPATH|/home/step|g" $STEPPATH/config/ca.json $STEPPATH/config/defaults.json

# 4. Seal all five, strict scope (guardrail 32), for the step-ca namespace.
kubeseal --controller-namespace sealed-secrets --controller-name sealed-secrets-controller \
  --fetch-cert > sealing-cert.pem

seal() { kubeseal --cert sealing-cert.pem --scope strict -o yaml; }

kubectl -n step-ca create secret generic step-certificates-certs \
  --from-file=root_ca.crt=$STEPPATH/certs/root_ca.crt \
  --from-file=intermediate_ca.crt=$STEPPATH/certs/intermediate_ca.crt \
  --dry-run=client -o yaml | seal
kubectl -n step-ca create secret generic step-certificates-config \
  --from-file=ca.json=$STEPPATH/config/ca.json \
  --from-file=defaults.json=$STEPPATH/config/defaults.json \
  --dry-run=client -o yaml | seal
kubectl -n step-ca create secret generic step-certificates-secrets \
  --from-file=root_ca_key=$STEPPATH/secrets/root_ca_key \
  --from-file=intermediate_ca_key=$STEPPATH/secrets/intermediate_ca_key \
  --dry-run=client -o yaml | seal
kubectl -n step-ca create secret generic step-certificates-ca-password \
  --from-file=password=ca_password.txt --dry-run=client -o yaml | seal
kubectl -n step-ca create secret generic step-certificates-provisioner-password \
  --from-file=password=provisioner_password.txt --dry-run=client -o yaml | seal

# Collect the five documents into infra/pki/step-ca-sealedsecrets.yaml, each carrying
#   argocd.argoproj.io/sync-wave: "2"   (after sealed-secrets at -2, before step-ca at 3)
```

## Record these three values for step-issuer

`infra/pki/step-clusterissuer.yaml` needs all three. Only the last is secret, and it is referenced rather than
written.

```bash
# caBundle — base64 of the root cert, single line, safe to commit
base64 -i $STEPPATH/certs/root_ca.crt | tr -d '\n'

# provisioner name and kid — public, safe to commit. Read them straight out of the ca.json
# you just generated; no running CA is needed.
jq -r '.authority.provisioners[] | select(.type=="JWK") | "\(.name)  \(.key.kid)"' $STEPPATH/config/ca.json
```

The `kid` is a thumbprint of the provisioner's public key, so it changes if the provisioner is recreated — if
certificates suddenly stop issuing after a CA rebuild, check the `kid` first.

## Destroy the plaintext

```bash
rm -rf "$STEPPATH" ca_password.txt provisioner_password.txt
```

Keep nothing but the sealed form. `root_ca.crt` is public and is already committed inside the `caBundle` field, so
there is no reason to retain a copy of the working directory.

## Recovery

There is no recovery path that preserves trust. If the root key is lost, the only option is a new CA and reissuing
every certificate under it. That is why this is the one step in the bootstrap with two verification gates in the
runbook.
