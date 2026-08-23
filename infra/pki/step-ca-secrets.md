# step-ca CA material — procedure, not material

This file documents how the CA secrets are produced. **It never contains key material** (guardrail 12).

## Why this is not automated

The `step-certificates` chart generates a fresh root CA when it does not find existing secrets. On a re-sync, a
re-install, or a namespace recreation that silently rotates the cluster's root of trust and invalidates every
certificate issued from it (guardrail 30). So the chart is configured with `inject.enabled: false` and pointed at
pre-existing secrets, and those secrets are committed here in sealed form.

## One-time bootstrap

Run once, on a trusted machine, before `infra/apps/step-certificates.yaml` first syncs.

```bash
# 1. Create the root and intermediate.
step certificate create "Platform Root CA" root_ca.crt root_ca.key \
  --profile root-ca --no-password --insecure
step certificate create "Platform Intermediate CA" intermediate_ca.crt intermediate_ca.key \
  --profile intermediate-ca --ca root_ca.crt --ca-key root_ca.key --no-password --insecure

# 2. Build the Secrets the chart expects. VERIFY the secret and key names against the values.yaml
#    of chart version 1.30.1 — they have moved between chart versions.
kubectl -n step-ca create secret generic step-certificates-secrets \
  --from-file=root_ca_key=root_ca.key \
  --from-file=intermediate_ca_key=intermediate_ca.key \
  --dry-run=client -o yaml > /tmp/step-ca-secrets.yaml

kubectl -n step-ca create secret generic step-certificates-ca-password \
  --from-literal=password='<generated>' --dry-run=client -o yaml >> /tmp/step-ca-secrets.yaml

# 3. Seal them for the step-ca namespace. Strict scope (guardrail 32) — this material must not be
#    unsealable anywhere else.
kubeseal --scope strict -o yaml < /tmp/step-ca-secrets.yaml > infra/pki/step-ca-sealedsecrets.yaml

# 4. Destroy the plaintext. Keep root_ca.crt — it is public and goes in the ClusterIssuer caBundle.
shred -u root_ca.key intermediate_ca.key /tmp/step-ca-secrets.yaml
```

## Record these three values for step-issuer

`infra/pki/step-clusterissuer.yaml` needs all three. Only the last is secret.

```bash
# caBundle — base64 of the root cert, single line, safe to commit
base64 -w0 root_ca.crt        # macOS: base64 -i root_ca.crt | tr -d '\n'

# provisioner name and kid — public, safe to commit. Read them back once the CA is running:
kubectl -n step-ca exec deploy/step-certificates -- step ca provisioner list \
  | python3 -c 'import json,sys; [print(p["name"], p["key"]["kid"]) for p in json.load(sys.stdin)]'

# provisioner password — SECRET. Seal it, strict scope, for the step-ca namespace:
kubectl -n step-ca create secret generic step-certificates-provisioner-password \
  --from-literal=password='<generated>' --dry-run=client -o yaml \
  | kubeseal --scope strict -o yaml >> infra/pki/step-ca-sealedsecrets.yaml
```

The `kid` is a thumbprint of the provisioner's public key, so it changes if the provisioner is
recreated — if certificates suddenly stop issuing after a CA rebuild, check the `kid` first.

## Prerequisites

- The sealed-secrets controller must be running (wave -2) before step 3.
- **The sealing key must already be backed up** (guardrail 31). Sealing before backing up the key means the material
  in Git is unrecoverable if the controller's key is lost.

## Recovery

There is no recovery path that preserves trust. If the root key is lost, the only option is a new CA and reissuing
every certificate under it. That is why this is the one step in the bootstrap with two verification gates in the
runbook.
