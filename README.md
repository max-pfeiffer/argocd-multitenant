# argocd-multitenant

GitOps source of truth for two ArgoCD instances — `argocd-infra` (cluster infrastructure) and
`argocd-multitenant` (three tenant teams) — on a vanilla Kubernetes cluster provisioned by Talos
Linux. See [CLAUDE.md](CLAUDE.md) for the architecture, the isolation model, and the guardrails.

This README is the **install runbook**: everything below is applied by a human, in order. Nothing
here is applied by ArgoCD, on purpose — see [guardrail 19](CLAUDE.md). Once the runbook is done,
the two root Applications reconcile `infra/` and `platform/` and there is nothing left to run by
hand.

---

## 1. Prerequisites (outside this repo)

These exist before any ArgoCD does. Verify each one; do not assume.

| Prerequisite | Check | Expected |
|---|---|---|
| Talos cluster reachable | `kubectl get nodes` | all `Ready` |
| Cilium as CNI (**not** Flannel) | `kubectl -n kube-system exec ds/cilium -- cilium version` | Cilium ≥ 1.20 |
| Cilium L2 announcements enabled | `kubectl -n kube-system get cm cilium-config -o jsonpath='{.data.enable-l2-announcements}'` | `true` |
| **Cilium ingressController disabled** | `kubectl -n kube-system get cm cilium-config -o jsonpath='{.data.enable-ingress-controller}'` | unset or `false` |
| Gateway API CRDs | `kubectl get crd gateways.gateway.networking.k8s.io` | present, v1.6.x |
| `TLSRoute` available | `kubectl get crd tlsroutes.gateway.networking.k8s.io -o jsonpath='{range .spec.versions[*]}{.name}={.served}{"\n"}{end}'` | `v1=true` |
| `cilium` GatewayClass accepted | `kubectl get gatewayclass` | `cilium … True` |
| NFS export reachable | `nc -z <nfs-server> 2049` | succeeds |

**Cilium's ingressController must be off.** This repo exposes everything through Gateway API and
creates no `Ingress` objects at all. With `ingressController.enabled=true`, Cilium stands up a
`cilium-ingress` LoadBalancer Service in `kube-system` that consumes one address from the
five-address pool below — leaving the allocation table one short. Turn it off in the Cilium
install (Talos machine config), not here.

Client-side tooling for this runbook: `kubectl` (kustomize is built in), `kubeseal`, `step`,
`helm` (only to inspect charts), `git`.

### The kubectl context

Every command below targets **`admin@argocd-multitenant`**. Confirm it rather than trusting
whatever is current — `bootstrap/operator/rendered.yaml` installs a cluster-wide operator with
broad RBAC, and `--server-side --force-conflicts` will take ownership of fields on any CRDs it
finds.

```bash
kubectl config use-context admin@argocd-multitenant
kubectl config current-context
```

---

## 2. DNS configuration (do this before step 3)

**Every hostname below must resolve before you install anything.** The addresses are static: each
`Gateway` pins its own with Gateway API's `spec.addresses`, out of the
`CiliumLoadBalancerIPPool` in [`infra/network/cilium-lb-ippool.yaml`](infra/network/cilium-lb-ippool.yaml)
(`192.168.20.245–249`). So the records can be created up front — there is nothing to wait for and
no address to look up afterwards.

Create the records in whatever serves the `.lan` zone (Pi-hole, dnsmasq, the router, an internal
BIND). All are `A` records; no PTR records are required.

| DNS name | Type | Address | Gateway | Namespace | Manifest | Serves |
|---|---|---|---|---|---|---|
| `argocd-infra-amt.lan` | A | 192.168.20.245 | `argocd-infra` | `argocd-infra` | [`install/argocd-infra-gateway.yaml`](install/argocd-infra-gateway.yaml) | infrastructure ArgoCD UI/API |
| `argocd-amt.lan` | A | 192.168.20.246 | `argocd` | `argocd-multitenant` | [`install/argocd-gateway.yaml`](install/argocd-gateway.yaml) | multi-tenant ArgoCD UI/API |
| `*.team-a-amt.lan` | A (wildcard) | 192.168.20.247 | `tenant` | `tenant-gateway` | [`platform/gateway/tenant-gateway.yaml`](platform/gateway/tenant-gateway.yaml) | team-a workloads (listener `team-a`) |
| `*.team-b-amt.lan` | A (wildcard) | 192.168.20.247 | `tenant` | `tenant-gateway` | [`platform/gateway/tenant-gateway.yaml`](platform/gateway/tenant-gateway.yaml) | team-b workloads (listener `team-b`) |
| `*.team-c-amt.lan` | A (wildcard) | 192.168.20.247 | `tenant` | `tenant-gateway` | [`platform/gateway/tenant-gateway.yaml`](platform/gateway/tenant-gateway.yaml) | team-c workloads (listener `team-c`) |
| `keycloak-amt.lan` | A | 192.168.20.248 | `keycloak` | `keycloak` | [`infra/apps/keycloak.yaml`](infra/apps/keycloak.yaml) | Keycloak (OIDC issuer for both instances) |
| — | — | 192.168.20.249 | — | — | — | spare, unallocated |

Notes:

- **The three tenant names share one address.** All three listeners live on the single `tenant`
  Gateway; the listener is selected by SNI/`Host`, not by IP. Per-team listeners (rather than one
  shared wildcard listener) are a boundary, not a style choice — see guardrail 24.
- **The tenant records must be wildcards.** Each listener's hostname is `*.team-<x>-amt.lan`, so a
  team's route hostname is always a label *under* their subdomain (`checkout.team-a-amt.lan`). The
  bare `team-a-amt.lan` is not covered by the listener and needs no record.
- **The wildcard is also what the certificate covers.** Each team's `Certificate` in
  `tenant-gateway` requests `*.team-<x>-amt.lan` from the `StepClusterIssuer`, so DNS and TLS agree
  by construction. Nothing here uses ACME challenges, so DNS does not have to be reachable from
  outside the LAN or expose an API.
- **`keycloak-amt.lan` is load-bearing for login on both instances.** It is the `issuer` in each CR's
  `oidcConfig` and the hostname Keycloak itself is configured with. A name that resolves differently
  for the browser and for the cluster breaks the OIDC redirect in a way that looks like an IdP
  misconfiguration.
- **Addresses are pinned with `spec.addresses`, never an annotation.** Cilium propagates no
  annotations from a `Gateway` onto the LoadBalancer Service it generates, so
  `io.cilium/lb-ipam-ips` on a Gateway is silently inert and LB-IPAM hands out whatever is free.
- **Onboarding a new team adds a DNS record**: a `*.team-<x>-amt.lan` wildcard pointed at the tenant
  gateway address, in the same reviewed PR as the listener, the certificate and the
  `NetworkPolicy` exception.

Verify DNS before continuing:

```bash
for h in argocd-infra-amt.lan argocd-amt.lan keycloak-amt.lan x.team-a-amt.lan \
         x.team-b-amt.lan x.team-c-amt.lan; do printf '%-24s ' "$h"; host "$h" | tail -1; done
```

---

## 3. Install the argocd-operator

One operator, cluster-wide, no OLM. This is the control plane's own definition and is never
applied by ArgoCD (guardrail 19).

```bash
# Render. The output is committed, so the PR diff *is* the change to the cluster.
kubectl kustomize bootstrap/operator > bootstrap/operator/rendered.yaml
git diff --stat bootstrap/operator/rendered.yaml     # expect: no drift on an unchanged pin

# Confirm the two settings that make the instances cluster-scoped.
grep -A1 -E 'WATCH_NAMESPACE|ARGOCD_CLUSTER_CONFIG_NAMESPACES' bootstrap/operator/rendered.yaml

# Dry-run against the live cluster, then apply. Server-side apply is not optional: the ArgoCD CRD
# is far larger than the 256 KiB annotation client-side apply depends on.
kubectl diff --server-side -f bootstrap/operator/rendered.yaml
kubectl apply --server-side --force-conflicts -f bootstrap/operator/rendered.yaml

kubectl wait --for=condition=Established crd/argocds.argoproj.io
kubectl -n argocd-operator rollout status deploy/argocd-operator-controller-manager
```

---

## 4. Bring up the `argocd-infra` instance

It goes first: it produces the StorageClass, the LB addresses, the certificate issuer and the
identity provider the other instance depends on.

**Apply it without `spec.oidcConfig`** — Keycloak does not exist yet, and an instance pointed at a
non-existent issuer is a confusing way to start.

```bash
kubectl create namespace argocd-infra --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace argocd-infra \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted --overwrite

sed '/^  oidcConfig: |/,/requestedScopes/d' install/argocd-infra-cr.yaml \
  | kubectl apply -f -

kubectl -n argocd-infra wait --for=condition=Ready pod --all --timeout=10m
kubectl -n argocd-infra get argocd argocd-infra -o jsonpath='{.status.phase}'   # Available
```

There is no way to reach it yet — no LB address, no certificate. Use a port-forward. Do **not**
work around it with `insecure: true` (guardrail 11):

```bash
kubectl -n argocd-infra port-forward svc/argocd-infra-server 8080:443
# initial admin password — note the secret is <cr-name>-cluster, not argocd-initial-admin-secret
kubectl -n argocd-infra get secret argocd-infra-cluster -o jsonpath='{.data.admin\.password}' | base64 -d
```

---

## 5. Break the sealing bootstrap cycle

A genuine chicken-and-egg, and the one place the runbook applies part of `infra/` by hand:

- `infra/` cannot sync until the CA material in `infra/pki/` is real, and
- the CA material cannot be sealed until the sealed-secrets controller is running, and
- the sealed-secrets controller is installed *by* `infra/`.

So bring up just those two layers directly. Both objects are byte-identical to what is in Git, so
`infra-root` adopts them on its first successful sync — this is a bootstrap shortcut, not a
permanent hand-managed resource.

```bash
kubectl apply -f infra/namespaces.yaml            # wave -3
kubectl apply -f infra/apps/sealed-secrets.yaml   # wave -2
kubectl -n sealed-secrets rollout status deploy/sealed-secrets-controller
```

### Back up the sealing key — before sealing anything

**Guardrail 31.** Lose this key and every `SealedSecret` in this repo *and in all three team repos*
becomes undecryptable at once; the only recovery is regenerating every secret from scratch.

```bash
mkdir -p ~/argocd-multitenant-backup && chmod 700 ~/argocd-multitenant-backup
kubectl -n sealed-secrets get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml \
  > ~/argocd-multitenant-backup/sealed-secrets-key-$(date +%Y%m%d).yaml
chmod 600 ~/argocd-multitenant-backup/*.yaml
```

Move that file into your secret manager and delete the local copy. It must not live next to the
repo indefinitely, and it must never be committed.

---

## 6. Bootstrap the CA and seal every secret

Full procedure, including the exact secret names the chart requires and why:
[`infra/pki/step-ca-secrets.md`](infra/pki/step-ca-secrets.md).

**This creates the cluster's root of trust exactly once.** Letting the chart generate a CA instead
means a re-sync silently rotates it and invalidates every certificate ever issued (guardrail 30).

Summary of what has to exist in Git before `infra/` can sync past wave 2:

| File | Contents |
|---|---|
| `infra/pki/step-ca-sealedsecrets.yaml` | root + intermediate keys, certs, `ca.json`/`defaults.json`, CA and provisioner passwords — five sealed objects |
| `infra/pki/step-clusterissuer.yaml` | `caBundle`, JWK provisioner `name` and `kid` from that CA |
| `infra/database/keycloak-db-sealedsecret.yaml` | Keycloak's Postgres owner credentials (`type: kubernetes.io/basic-auth`) |
| `infra/identity/oidc-client-sealedsecrets.yaml` | both OIDC client secrets, sealed for `keycloak` |
| `infra/identity/argocd-infra-oidc-sealedsecret.yaml` | the `argocd-infra` client secret, sealed for `argocd-infra` |
| `platform/argocd-oidc-sealedsecret.yaml` | the `argocd` client secret, sealed for `argocd-multitenant` |

Each OIDC client secret is sealed **twice from one plaintext** — once for `keycloak`, once for the
Argo CD namespace that consumes it. That duplication is the point: strict scoping (guardrail 32)
is what stops one namespace's sealed material being unsealed under another's. Never reach for
`--scope cluster-wide` to avoid it.

Destroy the plaintext when done, then commit and **push**:

```bash
git add -A && git commit -m "feat: CA bootstrap and sealed secrets"
git push origin main
```

> **The push is not optional.** Both root Applications sync
> `https://github.com/max-pfeiffer/argocd-multitenant` at `main`. Anything only committed locally
> is invisible to the cluster.

---

## 7. Hand over `infra/` to ArgoCD

The root app references an `AppProject` that lives inside the directory it syncs, so the project
goes first.

```bash
kubectl apply -f infra/projects/infra-project.yaml
kubectl apply -f bootstrap/infra-root-app.yaml

kubectl -n argocd-infra get applications -w
```

Sync waves −3 → 9 then bring up, in order: namespaces, sealed-secrets, the Cilium LB pool and L2
announcement policy, the NFS CSI driver, the default `StorageClass`, cert-manager, step-ca,
step-issuer, the `StepClusterIssuer`, CloudNativePG, both Keycloak operators, Keycloak's Postgres
cluster, Keycloak itself, and finally the realm, clients and groups.

**If the whole Application fails validation with `one or more synchronization tasks are not
valid`,** it is a resource whose CRD arrives in a later wave. Argo CD validates every resource in
an Application up front, so a single missing CRD fails every wave, including the one that would
install it. The fix is already applied to the affected resources —
`argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true` — and any new
infrastructure CR that introduces its own CRD needs the same annotation.

---

## 8. Expose both control planes

The LB pool now exists, so the two front doors can come up. They are hand-applied and deliberately
separate objects from the tenant gateway: a bad edit to a team listener must not be able to take
down the UI you would use to fix it (guardrail 19).

```bash
kubectl apply -f install/argocd-infra-gateway.yaml
kubectl apply -f install/argocd-gateway.yaml

kubectl get gateway -A -o custom-columns=\
NS:.metadata.namespace,NAME:.metadata.name,ADDR:.status.addresses[*].value
```

Each address must match the DNS table in section 2. If a Gateway sits at `Programmed=False` with
no address, the pool is exhausted — check whether something else has taken one:

```bash
kubectl get svc -A --field-selector spec.type=LoadBalancer
```

Verify HTTPS:

```bash
curl -sk https://argocd-amt.lan/api/version
curl -sk https://argocd-infra-amt.lan/api/version
```

### About the certificate

Both listeners are `mode: Passthrough`: `argocd-server` keeps `insecure: false` and terminates TLS
itself, so the gRPC the `argocd` CLI speaks needs no special handling at the gateway.

The certificate it presents is generated by the operator into the `argocd-tls` /
`argocd-infra-tls` secret, which carries an `ownerReference` back to the `ArgoCD` CR. **Do not
point a cert-manager `Certificate` at those secrets** — the operator reconciles them and the two
will fight. Consequences to be aware of:

- The cert is self-signed and its CN is `argocd-tls`, not the hostname, so browsers and `curl`
  will warn regardless of whether the step-ca root is trusted. `-k` is expected here.
- To get a properly trusted certificate, switch the listener to `mode: Terminate` with a
  cert-manager secret in the gateway namespace, an `HTTPRoute` to `argocd-server:443`, and a
  `BackendTLSPolicy` for the backend leg. What is **not** an option is `spec.server.insecure: true`
  (guardrail 11).

To trust the internal CA for everything *else* it issues (Keycloak, tenant workloads), install the
root certificate — it is public, and its base64 form is already committed in the `caBundle` field
of `infra/pki/step-clusterissuer.yaml`:

```bash
kubectl -n step-ca get secret step-certificates-certs -o jsonpath='{.data.root_ca\.crt}' \
  | base64 -d > root_ca.crt
# macOS: add to the login keychain and mark as trusted
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain root_ca.crt
```

---

## 9. Bring up the `argocd-multitenant` instance

Same shape as step 4, with one extra ordering constraint: **the operator cannot reconcile a
`spec.sourceNamespaces` entry for a namespace that does not exist yet**, and the team namespaces
are created by `platform-root`. So the CR is applied twice.

```bash
kubectl create namespace argocd-multitenant --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace argocd-multitenant \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted --overwrite

# First pass: no oidcConfig, no sourceNamespaces.
sed -e '/^  oidcConfig: |/,/requestedScopes/d' \
    -e 's/^  sourceNamespaces:$/  sourceNamespaces: []/' \
    -e '/^    - team-[abc]-apps$/d' install/argocd-cr.yaml | kubectl apply -f -

kubectl -n argocd-multitenant wait --for=condition=Ready pod --all --timeout=10m
```

Then hand `platform/` over to it, which creates the team namespaces:

```bash
kubectl apply -f platform/projects/platform-project.yaml
kubectl apply -f bootstrap/root-app.yaml
kubectl get ns team-a team-a-apps team-b team-b-apps team-c team-c-apps
```

Now re-apply the CR in full, so `sourceNamespaces` takes effect:

```bash
kubectl apply -f install/argocd-cr.yaml
```

Confirm the operator did its half of Applications-in-any-namespace — it labels each `-apps`
namespace and creates the `Role`/`RoleBinding` there. Never write these labels by hand
(guardrail 17):

```bash
kubectl get ns team-a-apps -o jsonpath='{.metadata.labels}'
# expect argocd.argoproj.io/managed-by-cluster-argocd: argocd-multitenant, and NO managed-by
kubectl -n team-a-apps get role,rolebinding
```

---

## 10. Wire up SSO, then disable the local admin

Keycloak's realm, clients and groups are created declaratively by the EDP operator from
`infra/identity/` — never in the admin console (guardrail 33).

What is *not* in Git is the users themselves. Create at least one user in the `platform` realm and
put them in the `platform-admins` group, then verify on both instances:

```bash
# Both CRs, now with oidcConfig, referencing the sealed client secrets.
kubectl apply -f install/argocd-infra-cr.yaml
kubectl apply -f install/argocd-cr.yaml

# The secrets Argo CD resolves $argocd-oidc:clientSecret / $argocd-infra-oidc:clientSecret from.
# The app.kubernetes.io/part-of=argocd label is what makes the lookup work.
kubectl -n argocd-multitenant get secret argocd-oidc --show-labels
kubectl -n argocd-infra get secret argocd-infra-oidc --show-labels
```

Log in via SSO at both hostnames, then confirm the token actually carries groups:

```bash
argocd account get-user-info --server argocd-amt.lan --grpc-web
```

**If login succeeds but everything is forbidden, the token has no `groups` claim.** Keycloak does
not emit one by default; the group-membership protocol mapper on each `KeycloakClient` is what adds
it. With `scopes: "[groups]"` and `defaultPolicy: ""`, a missing claim denies everything and reads
exactly like broken RBAC. Check the claim before touching the policy.

Once SSO works on both, close the local admin (guardrail 10):

```bash
# set spec.disableAdmin: true in BOTH CRs, then
kubectl apply -f install/argocd-infra-cr.yaml
kubectl apply -f install/argocd-cr.yaml
```

Only flip it back for break-glass recovery, and flip it off again afterwards.

---

## 11. Verify the boundaries

Run these once the install is complete. They check the things that fail *silently*.

```bash
# No namespace is claimed by both instances (guardrail 17).
kubectl get ns -L argocd.argoproj.io/managed-by

# The infra instance reconciles nothing outside its own namespace (guardrail 28).
kubectl get applications -A --field-selector metadata.namespace!=argocd-infra \
  -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name

# NetworkPolicy is genuinely ENFORCED, not merely accepted. This is the one that is
# indistinguishable from working if the cluster ever comes back up on Flannel (guardrail 20).
kubectl -n team-a run netpol-probe --rm -it --restart=Never --image=curlimages/curl -- \
  curl -sS --max-time 5 https://example.com
# Expected: times out. If it returns a page, every team namespace is wide open.

# No team can author Cilium policy or stand up its own gateway (guardrails 23, 25).
kubectl get cnp,ccnp -A                # expect: platform-owned only
kubectl get gateway -A                 # expect: argocd, argocd-infra, tenant, keycloak — nothing else
```

---

## Version pins

| Component | Version | Pinned in |
|---|---|---|
| argocd-operator | `9745824` (v0.18.0), image by digest | `bootstrap/operator/kustomization.yaml` |
| Argo CD | `v3.3.10` (both instances, never skewed) | `install/argocd-cr.yaml`, `install/argocd-infra-cr.yaml` |
| sealed-secrets | `v0.39.1` (chart 2.19.2) — a **release tag**, since the source is Git | `infra/apps/sealed-secrets.yaml` |
| csi-driver-nfs | `4.13.4` | `infra/apps/csi-driver-nfs.yaml` |
| cert-manager | `v1.21.1` | `infra/apps/cert-manager.yaml` |
| step-certificates | `1.30.1` | `infra/apps/step-certificates.yaml` |
| step-issuer | `1.11.0` | `infra/apps/step-issuer.yaml` |
| cloudnative-pg | `0.27.1` | `infra/apps/cloudnative-pg.yaml` |
| Keycloak operator | `26.7.2` | `infra/apps/keycloak-operator.yaml` |
| EDP Keycloak operator | `1.35.0` | `infra/apps/edp-keycloak-operator.yaml` |

Chart sources take a chart version in `targetRevision`; **Git sources take a release tag**. Mixing
those up produces `unable to resolve '<x>' to a commit SHA` at sync time.
