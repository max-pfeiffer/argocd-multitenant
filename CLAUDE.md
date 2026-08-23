# CLAUDE.md — ArgoCD Multi-Tenant Platform Repo

This repo is the GitOps source of truth for **two ArgoCD instances** on one **vanilla Kubernetes cluster provisioned by
Talos Linux** — not OpenShift, see "Cluster assumptions" below. Both are managed by the platform team:

- **`argocd-multitenant`** serves three teams (`team-a`, `team-b`, `team-c`). Most of this file is about it.
- **`argocd-infra`** installs cluster infrastructure: Sealed Secrets, Cilium LB config, NFS CSI, cert-manager, a step-ca
  CA, CloudNativePG, and Keycloak. It has no
  tenants, is never reachable by a team, and is installed **first** — the multi-tenant instance depends on what it
  provides.

The wall between them is the subject of its own section below, and of guardrails 27–30. Team application repos are separate and out of scope for this repo — this repo only contains
the ArgoCD install (via the **argocd-operator**), the per-team boundaries (`AppProject`, root `Application`, namespace
baseline), and RBAC/SSO config.

Read this whole file before making changes here. The "Guardrails" section at the bottom is non-negotiable: violating it
re-opens the exact privilege-escalation paths this setup exists to close.

## Cluster assumptions (vanilla Kubernetes on Talos Linux)

The target is **upstream Kubernetes provisioned by Talos Linux**. Much of the argocd-operator's documentation and most
blog posts about it assume OpenShift; do not copy those snippets verbatim. Nothing in this repo may use an OpenShift-only
API or rely on an OpenShift default. Concretely:

- **Gateway API, never `Route`, and not the operator's `Ingress` either.** The `ArgoCD` CR offers `spec.server.ingress`
  and `spec.server.route`; `route` is OpenShift-only (guardrail 22) and `ingress` is left disabled because this cluster
  exposes everything through Gateway API instead. See "Exposing ArgoCD and team workloads (Gateway API)".
- **No SecurityContextConstraints, no `oc`.** Pod-level hardening comes from Pod Security Admission, not SCCs. Every
  command in this file is plain `kubectl`; cluster-admin access comes from `talosctl kubeconfig`. **Every `kubectl`
  command in this file assumes the context `admin@argocd-multitenant`** — see below.
- **Talos enforces Pod Security Admission cluster-wide.** Talos configures the API server with a default
  `PodSecurityConfiguration` (`enforce: baseline`, `audit`/`warn: restricted`) that exempts only `kube-system`, so every
  namespace this repo creates is already at `baseline` whether or not it says so. We still label namespaces explicitly, so
  the level is reviewable in Git instead of inherited from a machine-config default that a Talos upgrade could change.
  Check the cluster's actual setting under `cluster.apiServer.admissionControl` in the Talos machine config before
  assuming the above.
- **Cilium is the CNI, so `NetworkPolicy` is genuinely enforced.** This matters more than it looks: Talos' *default* CNI
  is Flannel, which accepts `NetworkPolicy` objects and silently ignores them, making the `default-deny` policies in
  `platform/namespaces/` decorative. Cilium is what makes them real. Any cluster rebuild that lands back on the Talos
  default silently removes a tenant-isolation control — guardrail 20, with a verification recipe under "Common tasks".
- **Cilium brings its own policy CRDs, and they are a hole unless closed.** `namespaceResourceBlacklist` entries name a
  specific group/kind, so blocking `networking.k8s.io/NetworkPolicy` does nothing about `cilium.io/CiliumNetworkPolicy`.
  Every team `AppProject` therefore blacklists the whole `cilium.io` group — see the per-team `AppProject` section and
  guardrail 23.
- **A default `StorageClass` exists.** ArgoCD as configured here still requests no persistent volumes, but Redis HA and
  any bundled Prometheus/Grafana can have them when you want them. Name the class explicitly rather than leaning on the
  default marker — see the note under the `ArgoCD` CR.
- **Gateway API and Cilium's implementation of it are available**, so there is no separate ingress controller to run:
  Cilium provides the `cilium` `GatewayClass`. `GatewayClass` is cluster-scoped and therefore already out of reach of
  every team (`clusterResourceWhitelist: []`) — but `Gateway` itself is **namespaced**, so it needs an explicit blacklist
  entry. See guardrail 25.
- **cert-manager is not preinstalled.** Talos ships a deliberately minimal cluster. The argocd-operator itself needs no
  cluster prerequisites beyond this: it is installed directly from pinned upstream manifests, with no OLM in the picture.

### Prerequisites, applied before anything in `bootstrap/`

| Prerequisite | Status on this cluster | Notes |
|---|---|---|
| A `NetworkPolicy`-enforcing CNI | **present — Cilium v1.20.0** | installed in place of Talos' default Flannel (`cluster.network.cni.name: none` in the machine config); keep the kube-proxy vs. kube-proxy-replacement choice consistent cluster-wide |
| Gateway API CRDs | **present — v1.6.1** | Cilium provides the `cilium` `GatewayClass`; no separate ingress controller is needed. **Check which channel is installed** — `TLSRoute` ships only in the experimental channel and the ArgoCD front door below wants it: `kubectl get crd tlsroutes.gateway.networking.k8s.io` |
| Cilium L2 announcements **enabled in the Cilium install** | **verify** | `CiliumL2AnnouncementPolicy` is inert unless Cilium itself was installed with `l2announcements.enabled=true`. That is a value on the Cilium release, which lives outside this repo — the `argocd-infra` Application can create the policy but cannot turn the feature on |

Client-side tooling for the bootstrap steps: `kubectl` (with `kustomize` built in, or a standalone `kustomize` binary).
No OLM, no `operator-sdk`, no Helm.

These are genuinely external: Talos, Cilium and the Gateway API CRDs exist before any ArgoCD does. **Three things that
used to be listed here are now produced by the `argocd-infra` instance instead**, which is why it is installed first:

| Previously a prerequisite | Now provided by |
|---|---|
| A default `StorageClass` | `infra/apps/csi-driver-nfs` + `infra/storage/nfs-storageclass.yaml` |
| An address for `Gateway` LoadBalancer services | `infra/network/cilium-lb-ippool.yaml` (`192.168.20.245–249`) |
| cert-manager and a certificate issuer | `infra/apps/cert-manager` + `infra/apps/step-certificates` + `infra/apps/step-issuer` + the `StepClusterIssuer` |

Both versions above are load-bearing enough to record and re-check on upgrade, but nothing in this file depends on a
specific patch release: `kubectl -n kube-system exec ds/cilium -- cilium version` and
`kubectl get crd gateways.gateway.networking.k8s.io -o jsonpath='{.metadata.annotations}'` are the ground truth.

## Two ArgoCD instances, and the wall between them

One argocd-operator, two `ArgoCD` custom resources. They share nothing else.

| | `argocd-multitenant` | `argocd-infra` |
|---|---|---|
| Purpose | three tenant teams' workloads | cluster infrastructure |
| CR | `install/argocd-cr.yaml` | `install/argocd-infra-cr.yaml` |
| Root app | `bootstrap/root-app.yaml` → `platform/` | `bootstrap/infra-root-app.yaml` → `infra/` |
| `spec.sourceNamespaces` | the three team `-apps` namespaces | **empty** — every `Application` lives in `argocd-infra` |
| Cluster-scoped resources | forbidden to tenants (`clusterResourceWhitelist: []`) | permitted — that is its entire job |
| Who can log in | `platform-admins` + the three team groups | `platform-admins` only |
| Host | `argocd-amt.lan` | `argocd-infra-amt.lan` |
| Installed | second | **first** |

The isolation is not a matter of trust or naming. It rests on five mechanisms:

1. **Separate namespaces, separate CRs.** `argocd-cm`, `argocd-rbac-cm` and `argocd-secret` are per-instance and
   operator-managed. Neither instance can read or change the other's RBAC or SSO config.
2. **`argocd.argoproj.io/managed-by` is exclusive** (guardrail 17). A namespace is deployed into by exactly one instance.
   Infra namespaces carry `managed-by: argocd-infra`; team namespaces carry `managed-by: argocd-multitenant`. Neither
   instance is ever granted the other's namespaces.
3. **`argocd-infra` has no `sourceNamespaces` at all.** Applications-in-any-namespace is off for it, so no `Application`
   outside `argocd-infra` is reconciled by it — including anything a team could write.
4. **Disjoint Git paths.** `platform-root` syncs `platform/`; `infra-root` syncs `infra/`. Neither root app's path
   contains the other's. Note this is convention plus review, not enforcement — `AppProject.sourceRepos` restricts
   repositories, not paths. If you want it enforced, split `infra/` into its own repo and give the infra project only
   that repo URL.
5. **No shared identity.** No team group appears anywhere in the infra instance's `spec.rbac.policy`, and its
   `defaultPolicy` is `""` like the other's.

**What they do share is the operator.** One argocd-operator reconciles both CRs, so an operator upgrade moves both
instances at once. That is the residual coupling, and it is the reason operator upgrades stay gated behind a reviewed
render-and-apply (guardrail 13) rather than being automated.

Both namespaces are listed in `ARGOCD_CLUSTER_CONFIG_NAMESPACES`, so **both** instances run with a cluster-wide
`ClusterRole`. For `argocd-infra` that is the point. For `argocd-multitenant` it is what makes
Applications-in-any-namespace work, and it is why the `AppProject` boundaries — not the controller's own permissions —
are the tenant isolation. See guardrail 18: those two namespaces are the entire list, forever.

## Architecture summary

The rest of this file describes the **multi-tenant** instance unless it says otherwise; the infrastructure instance has
its own section further down.

- ArgoCD is installed and managed by the [argocd-operator](https://argocd-operator.readthedocs.io/en/latest/) (argoproj-labs), **not** by the `argo-helm` chart. The operator itself is installed cluster-wide from pinned upstream manifests via Kustomize — **no OLM** — and the ArgoCD instance is declared as a single `ArgoCD` custom resource.
- The `ArgoCD` CR is named `argocd` and lives in namespace **`argocd-multitenant`** (not the default `argocd`). That namespace is listed in the operator's `ARGOCD_CLUSTER_CONFIG_NAMESPACES` env var, which makes it a **cluster-scoped instance** — required for Applications-in-any-namespace, which the operator refuses to configure for a namespace-scoped instance.
- **The operator owns `argocd-cm`, `argocd-rbac-cm`, `argocd-cmd-params-cm` and `argocd-secret`.** Hand-edits to those ConfigMaps are reverted on the next reconcile. Every setting that used to be a Helm value or a ConfigMap patch is now a field on the `ArgoCD` CR (`spec.rbac.policy`, `spec.oidcConfig`, `spec.sourceNamespaces`, `spec.disableAdmin`, …), with `spec.extraConfig` as the escape hatch for `argocd-cm` keys the CRD does not expose.
- Auth is OIDC/SSO only, configured via `spec.oidcConfig`, pointed at the Keycloak that `argocd-infra` installs. **`spec.sso` stays unset** — that field makes the operator deploy and manage its own Keycloak or Dex, which is not what we want: our Keycloak is an infrastructure component with its own lifecycle, and Argo CD is just one of its OIDC clients. Local admin is disabled via `spec.disableAdmin: true` after bootstrap. IdP groups `team-a-devs` / `team-b-devs` / `team-c-devs` map to per-team, project-scoped ArgoCD roles; `platform-admins` maps to `role:platform-admin`.
- Each team owns one external Git repo for their app manifests. This repo does **not** contain team application code — only pointers to team repo URLs.
- **Self-service is Git-only, and teams author their own `Application` objects.** Each team gets a platform-created, platform-owned root `Application` ("app-of-apps") pointed at a directory in the team's repo (e.g. `app-of-apps/`). Everything the team commits there — literal `Application` manifests, including `project` and `destination` — becomes a live ArgoCD `Application`. Teams never touch the ArgoCD API/UI/CLI to create anything; they only ever `git push`.
- Because teams control the full `Application` spec, containment lives in the `AppProject` + namespace topology, not in a fixed template. See Guardrails.
- **Two namespaces per team** — this is an operator-specific requirement, see below:
  - `team-a-apps` — holds the team's `Application` CRs. Listed in the CR's `spec.sourceNamespaces`; the operator labels it `argocd.argoproj.io/managed-by-cluster-argocd: argocd-multitenant`.
  - `team-a` — the workload destination. Labelled `argocd.argoproj.io/managed-by: argocd-multitenant` by us.
- Every team `AppProject` blocks cluster-scoped resources (`clusterResourceWhitelist: []`), uses exact-match (non-wildcard) `sourceRepos`/`destinations`, and sets `sourceNamespaces: [<team>-apps]`.
- The built-in `default` AppProject is locked to empty `sourceRepos`/`destinations`/`clusterResourceWhitelist` so it can never be used as an escape hatch. The platform's own bootstrap app uses a dedicated `platform` AppProject instead.

Full rationale and background for these choices is in `docs/argocd-multi-team-setup-guide.md` (companion doc) if present in this repo — treat that as context, this file as the operative instructions.

## Repo layout (this repo)

```
.
├── CLAUDE.md
├── bootstrap/
│   ├── operator/
│   │   ├── kustomization.yaml      # pinned upstream base + image digest + patches
│   │   ├── namespace-patch.yaml    # PSA labels on the argocd-operator namespace
│   │   ├── manager-patch.yaml      # WATCH_NAMESPACE + ARGOCD_CLUSTER_CONFIG_NAMESPACES
│   │   └── rendered.yaml           # committed `kustomize build` output — the artifact that is applied
│   ├── root-app.yaml               # app-of-apps: argocd-multitenant manages this repo's /platform dir
│   └── infra-root-app.yaml         # app-of-apps: argocd-infra manages this repo's /infra dir
├── install/
│   ├── argocd-cr.yaml              # the multi-tenant ArgoCD CR — the whole instance config
│   ├── argocd-gateway.yaml         # Gateway + route for the multi-tenant UI — hand-applied (guardrail 19)
│   ├── argocd-infra-cr.yaml        # the infrastructure ArgoCD CR
│   └── argocd-infra-gateway.yaml   # Gateway + route for the infrastructure UI — hand-applied
├── infra/                          # synced by argocd-infra ONLY — never by platform-root
│   ├── projects/
│   │   ├── default-project.yaml    # the argocd-infra instance's own locked-down "default"
│   │   └── infra-project.yaml      # AppProject with clusterResourceWhitelist — the one place that exists
│   ├── namespaces.yaml             # every infra namespace, with PSA labels + managed-by
│   ├── network/
│   │   ├── cilium-lb-ippool.yaml   # CiliumLoadBalancerIPPool 192.168.20.245–249
│   │   └── cilium-l2-policy.yaml   # CiliumL2AnnouncementPolicy
│   ├── storage/
│   │   └── nfs-storageclass.yaml   # the default StorageClass, backed by the NFS CSI driver
│   ├── pki/
│   │   ├── step-ca-secrets.md      # how the CA key material is produced — NOT the material itself
│   │   ├── step-ca-sealedsecrets.yaml  # the CA material, sealed
│   │   └── step-clusterissuer.yaml # StepClusterIssuer — step-issuer against step-ca's JWK provisioner
│   ├── database/
│   │   └── keycloak-postgres.yaml  # CloudNativePG Cluster backing Keycloak
│   ├── identity/                   # EDP Keycloak operator CRs — realm config as Git objects
│   │   ├── realm.yaml
│   │   ├── client-argocd.yaml      # OIDC client for argocd-multitenant
│   │   ├── client-argocd-infra.yaml
│   │   └── groups.yaml             # platform-admins + team-<x>-devs
│   └── apps/                       # one Application per infrastructure component, sync-wave ordered
│       ├── sealed-secrets.yaml
│       ├── csi-driver-nfs.yaml
│       ├── cert-manager.yaml
│       ├── step-certificates.yaml
│       ├── step-issuer.yaml
│       ├── cloudnative-pg.yaml
│       ├── keycloak-operator.yaml
│       ├── edp-keycloak-operator.yaml
│       └── keycloak.yaml
└── platform/
    ├── gateway/
    │   └── tenant-gateway.yaml     # shared Gateway for team workloads, one listener per team
    ├── namespaces/
    │   ├── team-a.yaml             # team-a (workloads) + team-a-apps (Application CRs)
    │   │                           #   + ResourceQuota + default-deny NetworkPolicy + labels
    │   ├── team-b.yaml
    │   └── team-c.yaml
    ├── projects/
    │   ├── default-project.yaml    # locked-down built-in "default" AppProject
    │   ├── platform-project.yaml   # AppProject for this repo's own bootstrap app
    │   ├── team-a-project.yaml     # AppProject
    │   ├── team-b-project.yaml
    │   └── team-c-project.yaml
    └── root-apps/
        ├── team-a-root-app.yaml    # platform-owned app-of-apps Application per team
        ├── team-b-root-app.yaml
        └── team-c-root-app.yaml
```

There is deliberately **no `platform/config/` directory** any more: `argocd-cm`, `argocd-rbac-cm` and `argocd-cmd-params-cm`
are operator-managed and must not be shipped as standalone manifests. Their content lives in `install/argocd-cr.yaml`.

If this structure doesn't exist yet, create it — it's the layout all instructions below assume.
`docs/scaffolding-with-claude.md` is the runbook for doing that from scratch: the values to decide up front, the
phase-by-phase prompts, and — importantly — the bootstrap ordering, which is circular and easy to get wrong.

## The kubectl context

Everything applied by hand in this repo — the operator, the `ArgoCD` CR, the control-plane gateway, the one-time bootstrap
applies — goes to the **`admin@argocd-multitenant`** context:

```bash
kubectl config use-context admin@argocd-multitenant
kubectl config current-context          # confirm before every apply in this file
```

Prefer confirming the context over trusting whatever `current-context` happens to be. The manifests here are not
namespace-scoped nuisances if they land somewhere unintended: `bootstrap/operator/rendered.yaml` installs a cluster-wide
operator with broad RBAC, and `--server-side --force-conflicts` will take ownership of fields on any CRDs it finds. On a
cluster that already runs a different Argo CD, that is a bad afternoon. Where a command is destructive or hard to
reverse, pass `--context admin@argocd-multitenant` explicitly rather than relying on ambient state.

## Installing the argocd-operator

The operator is installed directly from upstream's Kustomize manifests, pinned, with **no OLM**. It runs cluster-wide in
its own `argocd-operator` namespace. This is a manual, platform-admin step — ArgoCD does not install its own operator
(guardrail 19).

The GitOps property we want here is not "ArgoCD applies it" — it deliberately does not — but **the cluster's operator
install is fully described by, and reconstructible from, this repo at a reviewed commit**. That means: pinned inputs, a
rendered output committed alongside them so the PR diff shows the real change, and an apply step that is a no-op when the
cluster already matches.

### `bootstrap/operator/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  # Pinned to an immutable commit SHA, not a tag. Tags can be moved; `?ref=v0.x.y` would then
  # resolve to different content on a later render with no diff in this file.
  - github.com/argoproj-labs/argocd-operator/config/default?ref=<pin-this-commit-sha>

namespace: argocd-operator          # renames upstream's `*-system` namespace and scopes every resource

images:
  - name: quay.io/argoprojlabs/argocd-operator
    digest: sha256:<pin-this-digest>    # digest, never a floating tag

patches:
  - path: namespace-patch.yaml
  - path: manager-patch.yaml
```

Upstream is a kubebuilder scaffold, so it applies its own `namePrefix: argocd-operator-` and its own `namespace:` before
our overlay's transformers run. **The consequence is that a resource's name at patch time is not the name you see in the
rendered output**, so matching a patch by name is unreliable in both directions — it can fail to match, and it can match
something you did not mean. Both patches therefore carry an explicit `target:` selector in `kustomization.yaml` and are
selected by *kind*, of which the base has exactly one each. The `metadata.name` in each patch body is documentation only.

Re-verify after any SHA bump that the selectors still hit exactly one resource, and that the container is still called
`manager`:

```bash
kubectl kustomize bootstrap/operator | grep -E '^kind:|^  name:|- name: manager'
```

### `bootstrap/operator/namespace-patch.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd-operator
  labels:
    pod-security.kubernetes.io/enforce: baseline      # Talos' cluster default, pinned explicitly (guardrail 21)
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### `bootstrap/operator/manager-patch.yaml` — this is where the instance becomes **cluster-scoped**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-operator-controller-manager
spec:
  template:
    spec:
      containers:
        - name: manager
          env:
            # Watch all namespaces. Nothing sets this for you any more: under OLM, the AllNamespaces
            # OperatorGroup injected WATCH_NAMESPACE="". Left at the pod's own namespace, the operator
            # quietly becomes namespace-scoped and never reconciles the ArgoCD CR in argocd-multitenant.
            - name: WATCH_NAMESPACE
              value: ""
            - name: ARGOCD_CLUSTER_CONFIG_NAMESPACES
              value: argocd-infra,argocd-multitenant   # exactly these two — see guardrail 18
```

### Render, review, apply

```bash
# 1. Render. The output is committed, so the PR diff *is* the change to the cluster.
kustomize build bootstrap/operator > bootstrap/operator/rendered.yaml

# 2. Confirm the two settings that matter actually made it in.
grep -A1 -E 'WATCH_NAMESPACE|ARGOCD_CLUSTER_CONFIG_NAMESPACES' bootstrap/operator/rendered.yaml

# 3. Dry-run against the live cluster before changing anything.
kubectl diff --server-side -f bootstrap/operator/rendered.yaml

# 4. Apply. Server-side apply is not optional: the ArgoCD CRD is far larger than the 256 KiB
#    last-applied-configuration annotation that client-side apply depends on.
kubectl apply --server-side --force-conflicts -f bootstrap/operator/rendered.yaml

kubectl wait --for=condition=Established crd/argocds.argoproj.io
kubectl -n argocd-operator rollout status deploy/argocd-operator-controller-manager
```

Commit `rendered.yaml` with every change to the kustomization, and have CI re-render and fail on drift:

```bash
kustomize build bootstrap/operator | diff -u bootstrap/operator/rendered.yaml - || {
  echo "rendered.yaml is stale — re-run kustomize build"; exit 1; }
```

Rendering to Git is what makes this reviewable. A reviewer looking at a bumped commit SHA learns nothing; a reviewer
looking at the rendered diff sees the new `ClusterRole` rules, the changed CRD schema, and the new image — which is
exactly the set of things worth catching on a control-plane upgrade.

### What OLM was doing, and what replaces it

Dropping OLM drops two things that were quietly working for you. Neither is hard to replace, but both have to become
somebody's job:

| OLM gave you | Replacement here |
|---|---|
| An `InstallPlan` approval gate before any operator upgrade | The upgrade *is* a PR: bump the SHA and digest, re-render, review, apply. Strictly better as an audit trail — it is in Git, not a `kubectl patch` someone ran |
| Generated operator RBAC you never looked at | The `ClusterRole` is now visible in `rendered.yaml`. Read RBAC changes in the upgrade diff; do not wave them through as noise |
| A refusal to upgrade a CRD in a way that invalidates existing CRs | Now on you. Before applying an operator upgrade, back up the CR (`kubectl -n argocd-multitenant get argocd argocd -o yaml > /tmp/argocd-cr-backup.yaml`) and check the rendered CRD diff for removed or newly-required fields |

The lost dependency resolution and channel subscription are not a loss here: there is one operator and its version is
pinned in Git either way.

`ARGOCD_CLUSTER_CONFIG_NAMESPACES` on the manager Deployment grants any ArgoCD instance in the listed namespaces a cluster-wide `ClusterRole` for
the application controller. That is what makes Applications-in-any-namespace and cross-namespace deploys work, and it is
also why the `AppProject` boundaries below are the real tenant isolation — the controller itself is cluster-powerful.
(`spec.defaultClusterScopedRoleDisabled: true` on the CR lets you substitute your own narrower `ClusterRole`; if you take
that route you own keeping it in sync with what the controller needs.)

## The ArgoCD instance (`install/argocd-cr.yaml`)

This single CR replaces the old `install/values.yaml` **and** the old `platform/config/*` ConfigMap patches. Apply it by
hand as a platform admin — it is deliberately *not* under `platform/`, so ArgoCD never self-manages its own RBAC and SSO
config (see guardrail 19):

```bash
kubectl create namespace argocd-multitenant --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace argocd-multitenant \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted --overwrite
kubectl apply -f install/argocd-cr.yaml
```

### Pod Security Admission for ArgoCD's own namespace

The control plane's namespace starts at `baseline` (Talos' cluster-wide default) rather than `restricted`, because whether
every operator-generated workload — server, controller, repo-server, redis, applicationset — carries a
`restricted`-compatible `securityContext` depends on the operator version, and a namespace flipped to `enforce: restricted`
rejects the offending pod outright at the worst possible moment. Tighten deliberately, using the `warn` level (already set
above) as the dry run:

```bash
kubectl -n argocd-multitenant rollout restart deploy    # warnings, if any, print here
kubectl label namespace argocd-multitenant pod-security.kubernetes.io/enforce=restricted --overwrite   # only if clean
```

If a component does need an exception, raise the level for that namespace and record why — never add a PSA exemption to
the Talos machine config, which would exempt the namespace from admission entirely and cluster-wide (guardrail 21).

```yaml
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: argocd
  namespace: argocd-multitenant
spec:
  version: v3.3.10                          # the version operator v0.18.0 targets — no skew
  resourceTrackingMethod: annotation        # required: multi-namespace app IDs are <namespace>/<name>
                                            #   and blow past the 63-char label-value limit
  disableAdmin: false                       # false only until OIDC is verified, then flip to true

  # Applications-in-any-namespace: exact team "-apps" namespaces only.
  # Never "*", never a glob ("team-*") or regex ("/^team-.*$/") — the operator expands both.
  sourceNamespaces:
    - team-a-apps
    - team-b-apps
    - team-c-apps

  oidcConfig: |
    name: Corp SSO
    issuer: https://keycloak-amt.lan/
    clientID: argocd
    clientSecret: $argocd-oidc:clientSecret
    requestedScopes: ["openid", "profile", "email", "groups"]

  rbac:
    defaultPolicy: ""                       # default-deny — never widen this
    scopes: "[groups]"
    policy: |
      p, role:platform-admin, applications, *, */*, allow
      p, role:platform-admin, projects, *, *, allow
      p, role:platform-admin, repositories, *, *, allow
      p, role:platform-admin, clusters, *, *, allow
      g, platform-admins, role:platform-admin

      p, role:team-a, applications, get, team-a/team-a-apps/*, allow
      p, role:team-a, applications, sync, team-a/team-a-apps/*, allow
      p, role:team-a, applications, action/*, team-a/team-a-apps/*, allow
      g, team-a-devs, role:team-a

      p, role:team-b, applications, get, team-b/team-b-apps/*, allow
      p, role:team-b, applications, sync, team-b/team-b-apps/*, allow
      p, role:team-b, applications, action/*, team-b/team-b-apps/*, allow
      g, team-b-devs, role:team-b

      p, role:team-c, applications, get, team-c/team-c-apps/*, allow
      p, role:team-c, applications, sync, team-c/team-c-apps/*, allow
      p, role:team-c, applications, action/*, team-c/team-c-apps/*, allow
      g, team-c-devs, role:team-c

  server:
    insecure: false                         # TLS terminates properly, no plaintext server mode
    replicas: 1
    host: argocd-amt.lan                # still set: the operator derives argocd-cm's `url` from it
    ingress:
      enabled: false                        # exposure is via Gateway API — see install/argocd-gateway.yaml
    # No `route:` block — that field is OpenShift-only (guardrail 22).

  controller:
    replicas: 1                             # raise once HA is needed
  repo:
    replicas: 1
  ha:
    enabled: false                          # flip on once you outgrow a single redis replica
```

The operator derives `argocd-cm`'s `url` key from `spec.server.host`/`spec.server.ingress`, so don't set it yourself —
but with `ingress.enabled: false` that derivation is worth checking rather than assuming, because a wrong `url` breaks the
OIDC redirect in a way that looks like an IdP problem:

```bash
kubectl -n argocd-multitenant get cm argocd-cm -o jsonpath='{.data.url}'   # expect https://argocd-amt.lan
```

If it is empty or the in-cluster service name, set it via `spec.extraConfig: {url: https://argocd-amt.lan}`. That is
a legitimate use of the escape hatch under guardrail 15 — with both `ingress` and `route` disabled the CRD exposes no
other way to state it.

Nothing in this CR requests persistent storage, which is deliberate rather than forced: a stock Argo CD needs none (Redis
is a cache — losing it costs a resync, not data). The cluster does have a default `StorageClass`, so anything that later
does want a PVC — `ha: true` with Redis persistence, a bundled Prometheus/Grafana — is straightforward to add. When you
add it, name the `storageClassName` explicitly instead of relying on the default marker: which class holds the
`is-default-class` annotation is cluster state that no one reviews, and a silent change to it would move ArgoCD's data
onto different storage on the next reprovision.
`spec.extraConfig` is available for `argocd-cm` keys the CRD doesn't expose — use it sparingly, and never for a key the CR
already models.

The `<project>/<namespace>/<application>` triple in `policy.csv` (e.g. `team-a/team-a-apps/*`) is required once
Applications live outside `argocd-multitenant`. The middle segment is the team's **`-apps`** namespace, not their workload
namespace — that's where the `Application` objects actually are.

Verify the instance came up, then log in with the initial admin password. Note the operator's secret name is
`<cr-name>-cluster`, not `argocd-initial-admin-secret`:

```bash
kubectl -n argocd-multitenant get pods
kubectl -n argocd-multitenant get secret argocd-cluster -o jsonpath='{.data.admin\.password}' | base64 -d
```

### The OIDC client secret

`argocd-secret` is operator-managed, so do not add keys to it. Instead create a separate secret carrying the Argo CD
part-of label and reference it with the `$<secret-name>:<key>` form (as `oidcConfig` above does). Populate it via your
secret manager — never commit it in plaintext to this repo:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-oidc
  namespace: argocd-multitenant
  labels:
    app.kubernetes.io/part-of: argocd     # required for Argo CD to resolve $argocd-oidc:clientSecret
stringData:
  clientSecret: <from your secret manager>
```

Confirm at least one `platform-admins` member can log in via SSO, then set `spec.disableAdmin: true` in
`install/argocd-cr.yaml` and re-apply.

## The infrastructure instance (`install/argocd-infra-cr.yaml`)

This instance exists to install the things a cluster needs before it can host tenants. It is applied by hand, before the
multi-tenant instance, into its own namespace:

```bash
kubectl create namespace argocd-infra --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace argocd-infra \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted --overwrite
kubectl apply -f install/argocd-infra-cr.yaml
```

```yaml
apiVersion: argoproj.io/v1beta1
kind: ArgoCD
metadata:
  name: argocd-infra
  namespace: argocd-infra
spec:
  version: v3.3.10                          # identical to the multi-tenant instance, always
  resourceTrackingMethod: annotation
  disableAdmin: false                       # true once Keycloak exists and OIDC is verified

  sourceNamespaces: []                      # NONE. Applications live only in argocd-infra (guardrail 28)

  oidcConfig: |                             # added after Keycloak is up — see the ordering below
    name: Corp SSO
    issuer: https://keycloak-amt.lan/realms/platform
    clientID: argocd-infra
    clientSecret: $argocd-infra-oidc:clientSecret
    requestedScopes: ["openid", "profile", "email", "groups"]

  rbac:
    defaultPolicy: ""                       # default-deny, same as the other instance
    scopes: "[groups]"
    policy: |
      p, role:infra-admin, applications, *, */*, allow
      p, role:infra-admin, projects, *, *, allow
      p, role:infra-admin, repositories, *, *, allow
      p, role:infra-admin, clusters, *, *, allow
      g, platform-admins, role:infra-admin
      # No team group appears here. Ever. (guardrail 29)

  server:
    insecure: false
    host: argocd-infra-amt.lan
    ingress:
      enabled: false                        # Gateway API, same as the other instance

  controller:
    replicas: 1
  repo:
    replicas: 1
  ha:
    enabled: false
```

The operator's initial admin secret for this instance is `argocd-infra-cluster` (it is `<cr-name>-cluster`), not
`argocd-cluster`.

**Chicken and egg: this instance comes up before there is any way to reach it.** cert-manager, the CA, and the
load-balancer address range are all things it is about to install. Until they exist, use a port-forward — do not
work around it by setting `insecure: true` or exposing it some other way:

```bash
kubectl -n argocd-infra port-forward svc/argocd-infra-server 8080:443
```

### The `infra` AppProject

`infra/projects/infra-project.yaml`. This is the **only** project anywhere in this repo with a non-empty
`clusterResourceWhitelist`, and it is deliberately unrestricted — installing CRDs, `StorageClass`es and `ClusterIssuer`s
is what this instance is for:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: infra
  namespace: argocd-infra
spec:
  description: Cluster infrastructure. Platform-only — no tenant ever has access to this project.
  sourceRepos:
    - https://github.com/max-pfeiffer/argocd-multitenant    # this repo, for infra/ manifests
    - https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
    - https://charts.jetstack.io
    - https://smallstep.github.io/helm-charts/
    - https://github.com/keycloak/keycloak-k8s-resources
    - https://epam.github.io/edp-helm-charts/stable
    - https://cloudnative-pg.github.io/charts
    - https://github.com/bitnami/sealed-secrets
  destinations:
    - server: https://kubernetes.default.svc
      namespace: "*"
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"                 # the documented exception — see below
```

Guardrail 3 says a team project never gets a cluster-scoped resource. That rule is not weakened here; this project is
simply not a team project. What makes the exception safe is everything in "the wall between them": the project lives in
an instance with no `sourceNamespaces`, no team group in its RBAC, and a separate hostname. **The moment any of those
three change, this whitelist becomes a cluster-admin escalation path.** Treat edits to this file and edits to
`install/argocd-infra-cr.yaml` as the same review.

`infra/projects/default-project.yaml` locks this instance's own built-in `default` project to empty, exactly as
`platform/projects/default-project.yaml` does for the other instance. Each ArgoCD instance auto-creates its own — locking
one does nothing for the other (guardrail 7).

### What it installs, and in what order

Ordering is expressed with `argocd.argoproj.io/sync-wave` annotations on the `Application` objects, because these have
real dependencies on each other:

| Wave | Application | Source | Why this wave |
|---|---|---|---|
| -2 | `sealed-secrets` | `https://github.com/bitnami/sealed-secrets` | **first, always** — every later wave that needs a secret gets it as a `SealedSecret`, and nothing can be unsealed before the controller exists |
| -1 | Cilium LB-IPAM pool + L2 announcement policy | `infra/network/` (this repo) | nothing with a `Gateway` gets an address until this exists |
| 0 | `csi-driver-nfs` | `https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts` | provides the CSI driver the `StorageClass` names |
| 1 | default `StorageClass` | `infra/storage/` (this repo) | step-ca's and Postgres' PVCs need a class |
| 2 | `cert-manager` v1.21.1 | `https://charts.jetstack.io` | its CRDs must exist before any `ClusterIssuer` |
| 3 | `step-certificates` v1.30.1 | `https://smallstep.github.io/helm-charts/` | the CA itself; needs storage (wave 1) and its sealed CA secrets (wave −2) |
| 4 | `step-issuer` v1.11.0 | `https://smallstep.github.io/helm-charts/` | the cert-manager external issuer; needs cert-manager's CRDs (2) |
| 5 | `StepClusterIssuer` | `infra/pki/` (this repo) | needs the step-issuer CRDs (4) *and* a running CA (3) |
| 5 | `cloudnative-pg` v0.27.1 | `https://cloudnative-pg.github.io/charts` | the Postgres operator Keycloak's database needs; independent of the PKI chain, so it shares wave 5 |
| 6 | Keycloak operator + EDP Keycloak operator | `https://github.com/keycloak/keycloak-k8s-resources`, `https://epam.github.io/edp-helm-charts/stable` | independent of each other; both must precede any Keycloak object |
| 7 | Postgres `Cluster` for Keycloak | `infra/database/` (this repo) | needs the CNPG operator (5) and storage (1) |
| 8 | `Keycloak` instance | `infra/apps/keycloak.yaml` | needs its database (7), a certificate (4) and the operator (6) |
| 9 | Realm, clients, groups | `infra/identity/` (this repo) | EDP CRs only reconcile against a running Keycloak |

### Identity: who does what

Three components, three distinct jobs, and mixing them up is the usual source of confusion:

| Component | Owns |
|---|---|
| **CloudNativePG** | the Postgres `Cluster` that backs Keycloak. The official Keycloak operator does not provision a database, and a `Keycloak` CR without one will not start |
| **Keycloak operator** (`keycloak.org`) | the Keycloak *server*: the `Keycloak` CR, its pods, its ingress/route wiring. Nothing about what is configured inside it |
| **EDP Keycloak operator** (`epam/edp-keycloak-operator`) | everything *inside* Keycloak, as Git objects: the realm, the OIDC clients, the groups, the role mappings |

The EDP operator is what closes the loop for this repo. Two distinct jobs it does here:

**1. Admin access to both ArgoCD instances.** A `KeycloakClient` for `argocd` and one for `argocd-infra`, plus the
`platform-admins` group that both instances' `spec.rbac.policy` maps to `role:platform-admin` / `role:infra-admin`.

**2. Team RBAC for the multi-tenant instance.** A `KeycloakRealmGroup` per team — `team-a-devs`, `team-b-devs`,
`team-c-devs` — matching the `g, <group>, role:<team>` lines in `install/argocd-cr.yaml`. The infrastructure instance has
no team groups and never will (guardrail 29); these exist purely to drive the other instance's RBAC.

**The one that will waste an afternoon:** Argo CD's `scopes: "[groups]"` needs a `groups` claim in the ID token, and
Keycloak does not emit one by default. The realm needs a group-membership protocol mapper on each client (EDP models this
on the `KeycloakClient`, or as a `KeycloakClientScope`). Without it, SSO login *succeeds*, the token carries no groups,
every policy line fails to match, and `defaultPolicy: ""` denies everything — which reads like broken RBAC rather than a
missing claim. Verify with `argocd account get-user-info` or by decoding the token before concluding the policy is wrong.

Because realm configuration lives in Git, the Postgres cluster is less precious than it looks: realms, clients and groups
can be reconciled back from `infra/identity/`. User sessions and any locally-created users cannot — configure CNPG
backups accordingly rather than assuming the DB is disposable.

Pin every chart with `targetRevision` on the `Application` — chart version for Helm sources, a release tag for Git ones
(guardrail 13). The `master` in the csi-driver-nfs URL is only where the chart *index* is served from; the chart version
you select with `targetRevision` is still immutable, so that URL is acceptable, but a Git source pointed at `master`
would not be.

### Cilium LB-IPAM and L2 announcements

`infra/network/cilium-lb-ippool.yaml` — five addresses, which is exactly enough, so allocate them deliberately:

```yaml
apiVersion: cilium.io/v2                 # v2alpha1 on older Cilium — check `kubectl api-resources | grep -i ippool`
kind: CiliumLoadBalancerIPPool
metadata:
  name: default-pool
spec:
  blocks:
    - start: "192.168.20.245"
      stop: "192.168.20.249"
```

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2
spec:
  loadBalancerIPs: true
  interfaces:
    - ^eth[0-9]+
  nodeSelector:
    matchLabels:
      kubernetes.io/os: linux
```

| Address | Assigned to |
|---|---|
| 192.168.20.245 | `argocd-infra` gateway |
| 192.168.20.246 | `argocd` (multi-tenant control plane) gateway |
| 192.168.20.247 | tenant gateway |
| 192.168.20.248 | Keycloak |
| 192.168.20.249 | spare |

One spare address. It was briefly consumed by an ACME HTTP-01 solver gateway; moving to step-issuer
removed the need for any solver endpoint at all, which is a second reason to prefer it here.

step-ca stays `ClusterIP` — cert-manager reaches it in-cluster, and an internal CA has no reason to be on the LAN.
Pin the assignments with `spec.addresses` / `loadBalancerIP` on the services rather than letting IPAM hand them out in
whatever order things come up, or a re-creation will shuffle them under your DNS records.

Remember `l2announcements.enabled=true` has to be set on the **Cilium install itself** — outside this repo. The policy
object above is silently inert without it, and the symptom is a `Gateway` service that has an IP but never answers ARP.

### Sealed Secrets: how both instances handle secrets

One `sealed-secrets` controller, installed by `argocd-infra` at wave −2, serves the whole cluster. It watches
`SealedSecret` objects in **any** namespace and produces the corresponding `Secret` alongside them. That makes it the
mechanism guardrail 12 has always pointed at, now concrete, and it is shared by both ArgoCD instances and by the teams:

| Who | Seals what | Lands in |
|---|---|---|
| `argocd-infra` | step-ca CA key material, Keycloak DB credentials, the `argocd-infra` OIDC client secret | `infra/` |
| `argocd-multitenant` | the `argocd` OIDC client secret | `platform/` |
| Teams | their own application secrets | their own repo, their own namespace |

**Sealing is namespace- and name-scoped, and it must stay that way** (guardrail 32). A `SealedSecret` sealed for
`team-a/db-password` cannot be unsealed anywhere else — moving the manifest to another namespace or renaming it makes it
undecryptable. For a multi-tenant cluster that is not a limitation, it is the point: a team can commit sealed material to
their own repo, and no other team can lift it, because the controller will refuse to unseal it under a different name or
namespace. The team `AppProject`'s `destinations` allowlist is the second lock — it stops a team-authored `SealedSecret`
from being *placed* in someone else's namespace in the first place.

The practical consequence when one secret is needed in two places — the OIDC client secret, which both the EDP
`KeycloakClient` and Argo CD's `$argocd-oidc:clientSecret` reference — is that you seal the same plaintext **twice**, once
per namespace, and commit both. Do not reach for `--scope cluster-wide` to avoid the duplication; that produces a secret
any namespace can unseal, which is exactly the property strict scoping exists to deny.

**The sealing key is the one piece of material that is not in Git and cannot be regenerated** (guardrail 31). Lose the
controller's private key and every `SealedSecret` in this repo, in `infra/`, and in all three team repos becomes
undecryptable at once — the recovery is regenerating every secret from scratch. Back it up out of band, before you seal
anything:

```bash
kubectl -n sealed-secrets get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > <your secret manager, not this repo>
```

### The CA: bootstrap step-ca from existing secrets

The one part of `infra/` where the GitOps default is actively dangerous. The `step-certificates` chart will happily
generate a fresh root CA if it does not find one — which means a re-sync, a re-install, or a namespace recreation
silently rotates your cluster's root of trust and invalidates every certificate issued from it.

So: **the CA key material is created once, out of band, and delivered as pre-existing secrets.** The chart is configured
not to generate anything (`inject.enabled: false` in the values, plus `existingSecrets`; verify the exact keys against
the values file for chart 1.30.1 — they have moved between versions).

Those secrets are `SealedSecret` objects committed under `infra/pki/`, which is why sealed-secrets is wave −2 and step-ca
is wave 3: the controller has to exist before the CA material can be unsealed. Generate the root and intermediate once
with `step certificate create`, seal each with `kubeseal` for the `step-ca` namespace, commit the sealed form, and destroy
the plaintext. `infra/pki/step-ca-secrets.md` documents which secrets must exist and how they were produced — the
procedure, never the material.

This is the payoff of putting sealed-secrets first: the CA bootstrap is a normal reviewed commit rather than a manual
`kubectl create secret` someone has to remember to run at exactly the right moment, and re-creating the cluster replays
it from Git.

Then the issuer, `infra/pki/step-clusterissuer.yaml`. **This is step-issuer, not ACME** — see below
for why:

```yaml
apiVersion: certmanager.step.sm/v1beta1
kind: StepClusterIssuer
metadata:
  name: step-ca
spec:
  url: https://step-certificates.step-ca.svc.cluster.local
  caBundle: <base64 of root_ca.crt>
  provisioner:
    name: <jwk-provisioner-name>
    kid: <jwk-provisioner-kid>
    passwordRef:
      name: step-certificates-provisioner-password
      namespace: step-ca
      key: password
```

**Why step-issuer rather than cert-manager's built-in ACME.** ACME was the obvious first choice and
it does not survive this environment. HTTP-01 validation makes step-ca fetch
`http://<hostname>/.well-known/acme-challenge/<token>`; with internal `.lan` names on per-hostname
LoadBalancer IPs, each hostname resolves to its *own* gateway, so no shared solver endpoint can ever
receive the challenge — and a per-gateway HTTP listener still could not issue the wildcard
certificates the tenant listeners need. DNS-01 would work but requires the `.lan` zone to expose an
API cert-manager supports. step-issuer sidesteps the question: cert-manager hands it a
`CertificateRequest` and it asks step-ca's JWK provisioner to sign. No challenge, no DNS dependency,
no port-80 reachability, wildcards included. It also removes a load-balancer address from the budget
— `.249` is spare again.

Every `certificateRefs` in this repo — both control-plane gateways, the Keycloak gateway, and the
tenant listeners — names this issuer with `kind: StepClusterIssuer, group: certmanager.step.sm`.
It is cluster-scoped because certificates are requested from four namespaces; a namespaced
`StepIssuer` would have to be duplicated into each.

## Exposing ArgoCD and team workloads (Gateway API)

Cilium implements Gateway API, so nothing here uses `Ingress`. There are **two** gateways on purpose, and the split is the
same one as everywhere else in this repo: the control plane's front door is hand-applied and never synced, while the
tenant front door is GitOps-managed.

| Gateway | Lives in | Managed by | Serves |
|---|---|---|---|
| `argocd` | `install/argocd-gateway.yaml`, namespace `argocd-multitenant` | platform admin, `kubectl apply` | the multi-tenant ArgoCD UI/API only |
| `argocd-infra` | `install/argocd-infra-gateway.yaml`, namespace `argocd-infra` | platform admin, `kubectl apply` | the infrastructure ArgoCD UI/API only |
| `tenant` | `platform/gateway/tenant-gateway.yaml`, namespace `tenant-gateway` | `platform-root` | team workloads, one listener per team |

Three gateways, three addresses out of the five in the pool. Both control-plane gateways follow the same shape and the
same reasoning; `install/argocd-infra-gateway.yaml` is `install/argocd-gateway.yaml` with the name, namespace, hostname
and backend service substituted.

They are separate objects, not two listeners on one gateway, so that a bad edit to a team listener cannot take down the
UI you would use to fix it — the same reasoning as guardrail 19. It costs one extra load-balancer address.

### ArgoCD's own front door

TLS **passthrough** is the recommended shape: `argocd-server` keeps `insecure: false` and terminates TLS itself, which
also means the gRPC the `argocd` CLI speaks needs no special handling at the gateway. Terminating at the gateway instead
drags in backend re-encryption *and* HTTP/2-to-backend config, and the usual "fix" people reach for is
`spec.server.insecure: true`, which guardrail 11 forbids.

```yaml
# install/argocd-gateway.yaml — applied by hand, right after install/argocd-cr.yaml.
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: argocd
  namespace: argocd-multitenant
spec:
  gatewayClassName: cilium
  listeners:
    - name: argocd-tls
      protocol: TLS
      port: 443
      hostname: argocd-amt.lan
      tls:
        mode: Passthrough                 # argocd-server presents its own cert
      allowedRoutes:
        namespaces:
          from: Same                      # nothing outside argocd-multitenant may attach
        kinds:
          - kind: TLSRoute
---
apiVersion: gateway.networking.k8s.io/v1alpha2   # TLSRoute is experimental-channel only
kind: TLSRoute
metadata:
  name: argocd-server
  namespace: argocd-multitenant
spec:
  parentRefs:
    - name: argocd
      sectionName: argocd-tls
  hostnames:
    - argocd-amt.lan
  rules:
    - backendRefs:
        - name: argocd-server
          port: 443
```

Two things to verify on the cluster before committing to this, because both are version- and operator-dependent:

1. **`TLSRoute` exists.** It ships only in Gateway API's experimental channel (see the prerequisites table). If the
   cluster has the standard channel only, either install the experimental channel or take the re-encrypt route below.
2. **Who owns the `argocd-server-tls` secret.** Argo CD serves the cert from that secret, but the operator may also
   generate and reconcile a self-signed one there. If it does, a cert-manager `Certificate` writing to the same name will
   fight the operator. Check with
   `kubectl -n argocd-multitenant get secret argocd-server-tls -o jsonpath='{.metadata.ownerReferences}'` before pointing
   cert-manager at it, and if the operator owns it, terminate at the gateway instead.

**Re-encrypt fallback**, if passthrough is unavailable: a `Gateway` listener with `protocol: HTTPS` / `mode: Terminate`
and a cert-manager-issued secret in the gateway namespace, an `HTTPRoute` to `argocd-server:443`, and a
`BackendTLSPolicy` giving the CA and hostname for the backend leg. Confirm Cilium honours `BackendTLSPolicy` at the
installed version, and expect to handle gRPC for the CLI explicitly. What is **not** an option is dropping to
`insecure: true` and routing plaintext.

### The tenant gateway

One listener per team, and the listener — not the team's own manifest — is what confines them:

```yaml
# platform/gateway/tenant-gateway.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-gateway
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tenant
  namespace: tenant-gateway
spec:
  gatewayClassName: cilium
  listeners:
    - name: team-a
      protocol: HTTPS
      port: 443
      hostname: "*.team-a-amt.lan"    # team-a can only ever claim names under its own subdomain
      tls:
        mode: Terminate
        certificateRefs:
          - name: team-a-apps-tls              # cert-manager, in THIS namespace — no ReferenceGrant needed
      allowedRoutes:
        namespaces:
          selector:
            matchLabels:
              kubernetes.io/metadata.name: team-a    # only team-a's workload namespace may attach
        kinds:
          - kind: HTTPRoute
          - kind: GRPCRoute
    # ... one identical block per team, with team-b / team-c substituted.
```

**Why per-team listeners rather than one wildcard listener with `from: Selector` over all teams:** Gateway API resolves
conflicting hostnames between routes by creation timestamp — oldest wins. On a shared listener, team-b committing an
`HTTPRoute` for `shop.apps-amt.lan` before team-a does simply takes the hostname, and if team-a already had it, a
later team can still attach overlapping path rules. Binding each team to a listener whose hostname is a wildcard under
*their own* subdomain removes the shared namespace entirely: there is nothing to race for. This is the Gateway API
analogue of exact-match `sourceRepos` — the boundary is in the platform-owned object, not in what the team writes.

A team then commits an `HTTPRoute` alongside their workload, in their own repo:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: checkout-service
  namespace: team-a                     # the workload namespace, not team-a-apps
spec:
  parentRefs:
    - name: tenant
      namespace: tenant-gateway
      sectionName: team-a               # their listener
  hostnames:
    - checkout.team-a-amt.lan  # must fall under the listener's wildcard
  rules:
    - backendRefs:
        - name: checkout-service
          port: 8080
```

Note the interaction with the `default-deny` `NetworkPolicy`: traffic from the gateway's Envoy to a team pod is ingress
that the team's own namespace denies by default. The narrow `NetworkPolicy` allowing it is platform-authored, like every
other policy exception (`NetworkPolicy` is blacklisted for teams) — budget for it during onboarding.

## Applications-in-any-namespace with the operator (and why teams get two namespaces)

This is the control that keeps team-authored `Application` objects out of the shared `argocd-multitenant` namespace, so
the blast radius of a mistake or a compromised repo is contained to that team's own turf instead of ArgoCD's control plane.

The operator drives it entirely from `spec.sourceNamespaces` on the CR — there is no `application.namespaces` param to set
and no `Role`/`RoleBinding` to write by hand. For each listed namespace the operator:

1. adds the label `argocd.argoproj.io/managed-by-cluster-argocd: argocd-multitenant` to the namespace,
2. creates a `Role` named `argocd_<namespace>` granting CRUD on `applications.argoproj.io` there, and
3. binds it to both the `argocd-server` and `argocd-application-controller` service accounts.

**The two labels are mutually exclusive, and this is why each team gets two namespaces.** A namespace that already carries
`argocd.argoproj.io/managed-by` (the label that lets ArgoCD *deploy* into it) is skipped by the reconciler above —
the operator logs `Skipping reconciling resources for namespace ... as it is already managed-by ...`, and will delete the
`managed-by-cluster-argocd` label if it was added earlier. The upstream docs call the combined case unsupported and point
at hand-written custom roles as the workaround. We don't take that workaround. Instead:

| Namespace     | Label                                                        | Set by       | Holds                       |
|---------------|--------------------------------------------------------------|--------------|-----------------------------|
| `team-a-apps` | `argocd.argoproj.io/managed-by-cluster-argocd: argocd-multitenant` | **operator** | the team's `Application` CRs |
| `team-a`      | `argocd.argoproj.io/managed-by: argocd-multitenant`          | this repo    | the team's workloads         |

Never put `managed-by` on an `-apps` namespace, and never hand-write `managed-by-cluster-argocd` anywhere — one wrong
label silently disables a team's self-service (or, worse, hands their namespace to a different instance).

The namespaces must exist **before** they are added to `spec.sourceNamespaces` — the operator errors out reconciling a
`sourceNamespaces` entry it can't `GET`. Sync `platform/namespaces/` first, then update the CR.

The other half of the boundary is unchanged and still lives in Argo CD itself: each team `AppProject` sets
`spec.sourceNamespaces: [team-a-apps]`. ArgoCD only lets an `Application` reference a project if the `Application`'s own
`metadata.namespace` is in that project's `sourceNamespaces`. Getting this list wrong is the single easiest way to reopen
cross-tenant access under this model — read the
[Applications-in-any-namespace docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/app-any-namespace/) before
touching it.

## Per-team namespace baseline

`platform/namespaces/team-a.yaml` (repeat per team, substituting the name):

```yaml
# Workload namespace: where the team's Applications actually deploy.
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    argocd.argoproj.io/managed-by: argocd-multitenant   # lets ArgoCD deploy here
    # Pinned explicitly rather than inherited from Talos' cluster default, so the level is reviewable
    # in Git. Raise to `restricted` per team once their workloads are known to comply.
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "50"
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: team-a
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
---
# Control namespace: holds ONLY the team's Application CRs.
# Deliberately carries NO argocd.argoproj.io/managed-by label — the operator adds
# argocd.argoproj.io/managed-by-cluster-argocd here from spec.sourceNamespaces, and the two
# labels are mutually exclusive.
apiVersion: v1
kind: Namespace
metadata:
  name: team-a-apps
  labels:
    # Nothing ever runs here (the quota below is pods: "0"), so this one can be `restricted` from day one.
    # PSA labels are unrelated to the argocd.argoproj.io/* label exclusivity described above — they are safe here.
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Belt and braces: the AppProject permits team-a-apps as a destination (the root app writes
# Application objects into it), so make sure nothing can be *run* there.
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-apps-no-workloads
  namespace: team-a-apps
spec:
  hard:
    pods: "0"
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: team-a-apps
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
```

**These policies are real because Cilium enforces them** — they would not be under Talos' default Flannel, which accepts
`NetworkPolicy` objects and ignores them, a state indistinguishable from "working" in `kubectl get netpol`. Enforcement is
a property of the cluster, not of the manifest, so verify it per cluster and after every CNI change — see "Common tasks".
Teams cannot punch holes in these themselves: both `networking.k8s.io/NetworkPolicy` and the entire `cilium.io` group are
in every team `AppProject`'s `namespaceResourceBlacklist`.

Note that a `default-deny` egress policy also blocks DNS. Each team's own `Application`s must ship the narrow
`Ingress`/`Egress` policies their workloads need (including egress to `kube-dns` in `kube-system`) — `NetworkPolicy` is
blacklisted in their `AppProject`, so those exceptions come through a platform PR, not a team commit. Budget for that
during onboarding rather than discovering it when the first workload cannot resolve anything.

Namespaces, quotas, and default-deny network policies are owned by this repo, not by team repos. The `Role`/`RoleBinding`
that used to live in this file is gone — the operator creates and owns that RBAC now; do not re-add it by hand. Every team
`Application` must always set `CreateNamespace=false` (see below) so a team can never create or alter a namespace via
their app repo.

## Per-team AppProject

`platform/projects/team-a-project.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd-multitenant
spec:
  description: Team A applications
  sourceRepos:
    - https://git.example.com/team-a/apps.git   # exact match — never a wildcard or prefix pattern
  destinations:
    - server: https://kubernetes.default.svc     # exact match — never "*"
      namespace: team-a                          # workloads
    - server: https://kubernetes.default.svc
      namespace: team-a-apps                     # the Application CRs the root app writes
  sourceNamespaces:
    - team-a-apps       # ONLY this team's own -apps namespace. Never argocd-multitenant, never another team's.
  clusterResourceWhitelist: []          # no cluster-scoped resources, ever
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota
    - group: networking.k8s.io
      kind: NetworkPolicy
    - group: cilium.io
      kind: "*"                         # the whole Cilium CRD group — see below
    - group: gateway.networking.k8s.io
      kind: Gateway                     # namespaced! clusterResourceWhitelist: [] does NOT cover it
    - group: gateway.networking.k8s.io
      kind: TLSRoute                    # L4 exposure — teams get HTTPRoute/GRPCRoute only
    - group: gateway.networking.k8s.io
      kind: TCPRoute
    - group: gateway.networking.k8s.io
      kind: UDPRoute
    - group: gateway.networking.k8s.io
      kind: ReferenceGrant              # cross-namespace backendRefs stay platform-reviewed
    - group: argoproj.io
      kind: AppProject                  # teams author Applications, never projects
  permitOnlyProjectScopedClusters: true   # Applications can't target a cluster not registered to this project
  orphanedResources:
    warn: true
```

`sourceRepos` and `destinations` being exact, non-wildcard matches is what actually stops a team from escaping their lane —
even if a team-authored `Application` sets `project: team-b`, sync fails because team-b's project doesn't permit team-a's
repo URL or namespaces. The project name in the manifest is not the trust boundary; the repo+destination allowlist is.

The `cilium.io` blacklist entry closes a gap that only exists because the CNI is Cilium. `namespaceResourceBlacklist`
matches on group and kind, so an entry for `networking.k8s.io/NetworkPolicy` says nothing about
`cilium.io/CiliumNetworkPolicy` — and Cilium enforces both. Cilium policies are additive allows, so a team-authored
`CiliumNetworkPolicy` in their own namespace would add exceptions on top of the platform's `default-deny` without ever
touching the platform's objects. The same reasoning covers the rest of the group: `CiliumEnvoyConfig` can insert L7
redirection, `CiliumLocalRedirectPolicy` can steer traffic. Blacklisting `kind: "*"` for the group rather than
enumerating kinds is deliberate — a Cilium upgrade that adds a namespaced CRD is then closed by default instead of open
until someone notices. Cluster-scoped Cilium kinds (`CiliumClusterwideNetworkPolicy`, `CiliumEgressGatewayPolicy`) are
already unreachable via `clusterResourceWhitelist: []`; this entry is about the namespaced ones.

The `gateway.networking.k8s.io` entries are enumerated rather than wildcarded, because unlike `cilium.io` this group has
kinds teams genuinely need: `HTTPRoute` and `GRPCRoute` are the whole point of tenant self-service exposure. What they do
not get is `Gateway` — which is **namespaced**, so `clusterResourceWhitelist: []` does not stop a team from standing up
their own gateway (and, with a `GatewayClass` present, their own load balancer) entirely outside the tenant listener
model. The L4 route kinds are belt and braces: the tenant listener already restricts attachment to
`kinds: [HTTPRoute, GRPCRoute]`, so a `TLSRoute` could not attach today, but blacklisting them means a future listener
cannot silently widen what teams can already have created. `ReferenceGrant` is blocked because it is the object that
authorises a route in one namespace to reach a `Service` in another — cross-tenant backends should be a platform decision,
not a self-served one. Because this list is an enumeration, **re-review it whenever Gateway API is upgraded**
(guardrail 25); a new namespaced kind in the group is open until it is added here.

### Locked-down `default` AppProject

`platform/projects/default-project.yaml` — Argo CD auto-creates a `default` project with `sourceRepos: ["*"]` and
`destinations: [{server: "*", namespace: "*"}]`. Left as-is, it's the easiest escalation path: any team-authored
`Application` that sets `project: default` gets an unrestricted deploy target. Lock it to empty on every environment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
  namespace: argocd-multitenant
spec:
  description: Locked down — not for use. Do not reference from any Application.
  sourceRepos: []
  destinations: []
  clusterResourceWhitelist: []
```

### `platform` AppProject

`platform/projects/platform-project.yaml` — the project for this repo's own bootstrap app, so nothing platform-owned ever
needs an exception to the locked-down `default` above. Its power is bounded by the single exact `sourceRepos` entry, and no
team role is ever granted access to it:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd-multitenant
spec:
  description: Platform-owned config from the argocd-multitenant repo
  sourceRepos:
    - https://github.com/max-pfeiffer/argocd-multitenant   # exact match, this repo only
  destinations:
    - server: https://kubernetes.default.svc
      namespace: "*"                  # platform-owned: creates team namespaces and control-plane objects
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
```

The `namespace: "*"` here is the one legitimate wildcard destination in this repo. It is acceptable **only** because this
project is reachable from exactly one repo that the platform team controls. Never copy this shape into a team project.

## Per-team root Application ("app-of-apps", team-authored self-service)

`platform/root-apps/team-a-root-app.yaml` — platform-owned and platform-edited only. This is the *only* ArgoCD object the
platform team manages per team going forward; everything under the path it watches is authored by the team themselves.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: team-a-root
  namespace: team-a-apps     # the team's control namespace, not argocd-multitenant
spec:
  project: team-a                       # fixed — never templated from repo content
  source:
    repoURL: https://git.example.com/team-a/apps.git
    targetRevision: main
    path: app-of-apps                   # team commits raw Application manifests under this path
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: team-a-apps              # the Application objects land here
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

Everything the team commits under `app-of-apps/` in their own repo is a literal `Application` manifest they write
themselves — for example `app-of-apps/checkout-service.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: checkout-service
  namespace: team-a-apps                # must be team-a-apps — enforced by sourceNamespaces, not by trust
spec:
  project: team-a                       # must be team-a — enforced by sourceRepos/destinations, not by trust
  source:
    repoURL: https://git.example.com/team-a/apps.git
    targetRevision: main
    path: apps/checkout-service
  destination:
    server: https://kubernetes.default.svc
    namespace: team-a                   # workloads go to the workload namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

Nothing here relies on the team writing `project: team-a` and `namespace: team-a-apps` correctly out of good faith — if
they wrote something else, the `AppProject`'s `sourceRepos`/`destinations`/`sourceNamespaces` reject it. Treat any
`ComparisonError` caused by a project/destination mismatch as a signal worth looking at (mistake, or someone probing the
boundary), not just a noisy sync failure to dismiss.

Require branch protection and `CODEOWNERS` on every team repo's `app-of-apps/` directory — with the template gone, PR
review is now doing real work as a first line of defense, on top of (not instead of) the AppProject enforcement above.

## Bootstrapping this repo via app-of-apps

Once the operator is running and `install/argocd-cr.yaml` has been applied, apply `bootstrap/root-app.yaml` once, by hand,
as the platform admin — after that, ArgoCD manages this repo's own `platform/` directory the same way it manages team apps:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-root
  namespace: argocd-multitenant
spec:
  project: platform                     # dedicated platform project, never "default"
  source:
    repoURL: https://github.com/max-pfeiffer/argocd-multitenant
    targetRevision: main
    path: platform
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd-multitenant
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

`bootstrap/infra-root-app.yaml` is the same object for the other instance: `metadata.namespace: argocd-infra`,
`project: infra`, `path: infra`, `destination.namespace: argocd-infra`. Apply it by hand once, in the same way.

Chicken-and-egg note: each root app references an AppProject that lives inside the directory it syncs. Apply
`platform/projects/platform-project.yaml` (respectively `infra/projects/infra-project.yaml`) by hand once, immediately
before its root app; from then on ArgoCD owns it.

### Order of the whole bootstrap

The infrastructure instance produces things the multi-tenant instance needs — a storage class, an LB address, a
certificate issuer, an identity provider — so it goes first, and it comes up without any of them:

1. Talos, Cilium (with `l2announcements.enabled=true`), Gateway API CRDs. Outside this repo.
2. `bootstrap/operator/` — one operator, both namespaces in `ARGOCD_CLUSTER_CONFIG_NAMESPACES`.
3. `argocd-infra` namespace + `install/argocd-infra-cr.yaml`. Reach it by `kubectl port-forward`; there is no LB address
   or certificate yet.
4. `infra/projects/infra-project.yaml`, then `bootstrap/infra-root-app.yaml`. Sync waves −1→7 bring up LB-IPAM, NFS CSI,
   the default `StorageClass`, cert-manager, step-ca, step-issuer and its `StepClusterIssuer`, then Keycloak.
5. `install/argocd-infra-gateway.yaml` — now there is an address and a certificate, so the port-forward can stop.
6. Keycloak realm and OIDC clients (`argocd`, `argocd-infra`) via the EDP operator, from `infra/`.
7. `argocd-multitenant` namespace + `install/argocd-cr.yaml` + `install/argocd-gateway.yaml`.
8. Team namespaces → `spec.sourceNamespaces` → `platform/projects/platform-project.yaml` → `bootstrap/root-app.yaml`,
   in that order (see "Common tasks").
9. Verify SSO on both instances, then `spec.disableAdmin: true` on both CRs.

Steps 3–6 are the part that does not appear anywhere else in this file, because they only happen once.

`bootstrap/operator/` and `install/` stay outside the synced path on purpose. The operator manifests and the `ArgoCD` CR
are the control plane's own definition — including its RBAC policy and SSO config — and are applied by a platform admin
(by hand or from a reviewed CI job), never by ArgoCD itself. A self-managed control plane can lock you out of the tool you'd
use to fix it: an operator upgrade that breaks reconciliation, applied by the operator's own ArgoCD, leaves you with no
working ArgoCD to roll it back with.

This is not a gap in the GitOps model, it is where the model's boundary sits. Everything is still declared in Git, pinned,
reviewed, and applied from a rendered artifact — the reconciler for that one layer is a person or a CI job rather than
ArgoCD. `kubectl diff --server-side -f bootstrap/operator/rendered.yaml` is the drift check that stands in for continuous
reconciliation; run it from CI on a schedule if you want the drift signal without the self-management risk.

## Common tasks

**Add a new team:** in one reviewed PR, add `platform/namespaces/<team>.yaml` (workload namespace with the `managed-by`
label + PSA labels + quota + netpol, plus the `<team>-apps` control namespace with the `pods: "0"` quota and its own PSA
labels),
`platform/projects/<team>-project.yaml` (exact-match `sourceRepos`/`destinations`, `sourceNamespaces: [<team>-apps]`,
`permitOnlyProjectScopedClusters: true`), and `platform/root-apps/<team>-root-app.yaml`. Then, in `install/argocd-cr.yaml`,
add `<team>-apps` to `spec.sourceNamespaces` and a new `role:<team>` block to `spec.rbac.policy` using the
`<team>/<team>-apps/*` pattern. **Also add a `KeycloakRealmGroup` for `<team>-devs` under `infra/identity/`** — the group
named in `spec.rbac.policy` has to exist in Keycloak, and it is created there declaratively, not in the console. That
part of the change is synced by the *other* ArgoCD instance, so a team onboarding PR now spans `platform/`, `install/`
and `infra/`. If the team needs inbound traffic, the same PR adds their listener to
`platform/gateway/tenant-gateway.yaml` (hostname under their own subdomain, `allowedRoutes` selector pinned to their
workload namespace) plus the certificate and `NetworkPolicy` exception that go with it.

Order matters: let `platform-root` sync the namespaces **first**, then `kubectl apply -f install/argocd-cr.yaml`. The
operator cannot reconcile a `sourceNamespaces` entry for a namespace that doesn't exist yet. It's a security-boundary
change — review it like one.

**Verify a team's wiring before granting access:** confirm the operator labelled the control namespace
(`kubectl get ns <team>-apps -o jsonpath='{.metadata.labels}'` should show `managed-by-cluster-argocd: argocd-multitenant`
and no `managed-by`), then push a placeholder `Application` manifest into the team's repo `app-of-apps/` folder, confirm it
lands in `<team>-apps` under the right project and deploys into `<team>`, then add the team's SSO group.

**Change any ArgoCD setting:** edit `install/argocd-cr.yaml` and re-apply. Never `kubectl edit` `argocd-cm`,
`argocd-rbac-cm`, `argocd-cmd-params-cm` or `argocd-secret` — the operator reverts them, usually right after you've
convinced yourself the change worked.

**Upgrade Argo CD:** bump `spec.version` in **both** `install/argocd-cr.yaml` and `install/argocd-infra-cr.yaml` — they must not drift apart — and re-apply each. Check the operator's compatibility matrix first: the current pin, v3.3.10, is what operator v0.18.0 targets, so moving one without the other reintroduces skew. **Upgrade the operator:** bump the
`?ref=` commit SHA and the image digest in `bootstrap/operator/kustomization.yaml`, re-render `rendered.yaml`, review the
diff (RBAC and CRD changes especially), then `kubectl apply --server-side --force-conflicts`. Back up the `ArgoCD` CR
first — nothing gates a breaking CRD change for you now. Do the two upgrades separately, and check the operator's
compatibility matrix before either.

**Expose a team workload:** the team commits an `HTTPRoute` in their own repo; the platform side is a listener on the
tenant gateway (`platform/gateway/tenant-gateway.yaml`), a cert-manager `Certificate` for that team's wildcard hostname in
the `tenant-gateway` namespace, a DNS record, and the `NetworkPolicy` exception allowing gateway→pod ingress. All four are
platform-owned and belong in the same reviewed PR that onboards the team.

**Rotate a secret:** re-seal it with `kubeseal`, commit, let ArgoCD sync. The unsealed `Secret` is owned by the
controller, so do not `kubectl edit` it — the change survives until the next reconcile and then vanishes, which is a
confusing hour. If the secret exists in two namespaces (the OIDC client secret does), re-seal both.

**Add an infrastructure component:** a new `Application` under `infra/apps/`, in the `infra` project, with a
`targetRevision` pinned to a chart version or release tag and an `argocd.argoproj.io/sync-wave` that reflects its real
dependencies. If it needs a load-balancer address, take one from the pool table and check the pool is not exhausted
first. It goes in the `argocd-infra` namespace like every other Application on that instance — never in a team namespace,
never in `argocd-multitenant`.

**Verify the two instances are not overlapping:**

```bash
# No namespace is claimed by both instances
kubectl get ns -L argocd.argoproj.io/managed-by
# The infra instance reconciles nothing outside its own namespace
kubectl -n argocd-infra get applications
kubectl get applications -A --field-selector metadata.namespace!=argocd-infra   -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name   # expect: only team -apps + argocd-multitenant
```

**Verify `NetworkPolicy` is actually enforced** (once per cluster, and again after any CNI change or cluster rebuild — a
cluster that comes back up on Talos' default Flannel silently loses enforcement while every manifest still looks right):

```bash
# 1. Cilium's own view: the endpoint should report policy enforcement enabled in both directions.
kubectl -n kube-system exec ds/cilium -- cilium status --brief
kubectl -n kube-system exec ds/cilium -- cilium endpoint list | grep team-a

# 2. End-to-end, which is the one that actually proves it.
kubectl -n team-a run netpol-probe --rm -it --restart=Never --image=curlimages/curl -- \
  curl -sS --max-time 5 https://example.com
# Expected: the default-deny egress policy makes this time out.
# If it returns a page, the CNI is not enforcing NetworkPolicy and every team namespace is wide open.
```

(That probe assumes the namespace is at `enforce: baseline`; under `restricted` the pod needs an explicit
`securityContext`, and it counts against the namespace `ResourceQuota` while it runs.)

**Confirm no team can author Cilium policy or their own gateway:**

```bash
kubectl get cnp,ccnp -A                       # expect: platform-owned objects only
kubectl get gateway -A                        # expect: exactly `argocd` and `tenant`
kubectl get httproute,grpcroute -A            # team routes: only in their own workload namespace
```

Anything in a `team-*` namespace from the first two means that team's `AppProject` is missing a blacklist entry.

**Check what Pod Security Admission would reject** before tightening a namespace: set
`pod-security.kubernetes.io/warn=restricted` on it, redeploy, and read the warnings. Never widen a level to make a
workload fit without recording why.

**Rotate/onboard SSO groups:** done in the IdP, not in this repo — this repo only references group names in
`spec.rbac.policy`.

## Guardrails — do not violate these when editing this repo

1. `spec.rbac.defaultPolicy` on the `ArgoCD` CR stays `""` (default-deny). Never widen it to `role:readonly` or broader.
2. No team role gets a wildcard resource, action, or a permission outside `applications` in their own project. Teams never get `applications, create` — self-service happens via Git, not the ArgoCD API.
3. Every team `AppProject` keeps `clusterResourceWhitelist: []`. If a team needs a cluster-scoped resource or CRD, that's a reviewed, explicit exception added here by the platform team — never a default.
4. Every team `Application` keeps `syncOptions: [CreateNamespace=false]`. Namespace lifecycle stays platform-owned.
5. Every team `AppProject`'s `sourceRepos` and `destinations` are exact-match values — never `"*"`, never a prefix/glob pattern. This is the actual boundary that contains a team-authored `Application`, regardless of what `project`/`destination` the team writes into it. (The `platform` project's `namespace: "*"` destination is the sole, platform-owned exception.)
6. Every team `AppProject`'s `sourceNamespaces` contains **only that team's own `-apps` namespace** — never `argocd-multitenant`, never a workload namespace, never another team's. Getting this wrong is the specific, documented way to reopen cross-tenant access under Applications-in-any-namespace.
7. The built-in `default` AppProject stays locked to empty `sourceRepos`/`destinations`/`clusterResourceWhitelist` — **in both instances**. Each ArgoCD instance auto-creates its own `default`; locking one does nothing for the other. Platform-owned apps use the dedicated `platform` or `infra` project, not an exception to this rule.
8. `permitOnlyProjectScopedClusters: true` on every team `AppProject`.
9. Team repo credentials (`Repository`/`argocd-repo-creds` secrets) are always scoped to their owning `AppProject` via the `project` field — never registered project-wide.
10. `spec.disableAdmin` stays `true` once OIDC is verified. Only flip it back for break-glass recovery, and flip it off again afterward.
11. `spec.server.insecure` stays `false`; ArgoCD is always reached over TLS. This holds however it is exposed — if a routing setup is awkward to terminate correctly, fix the routing, never drop the server to plaintext.
12. Secrets are committed only in sealed form, anywhere in this repo or a team repo. Sealed Secrets is the mechanism — a `SealedSecret` in Git, never a plaintext `Secret`, and never a value pasted into a values block. The OIDC client secrets live in their own labelled `Secret`s produced by unsealing, never in `argocd-secret`.
13. Pin every version explicitly: the operator manifests via a `?ref=<commit-sha>` on the Kustomize base and its image via `sha256:` digest, and Argo CD via `spec.version` on both CRs, which must always carry the identical value. `spec.version` takes an exact release tag or, for a stronger pin, a digest — the operator composes any value containing `:` as `image@sha256:…`. An exact tag is the floor; prefer the digest where the extra opacity is worth it, and record the other form in a comment either way. Never a floating tag, never a branch ref, never `latest`. Nothing that a re-render or a re-push could quietly change underneath you.
14. The per-team root `Application` (`platform/root-apps/<team>-root-app.yaml`) is platform-owned — teams commit `Application` manifests inside the path it watches, they never edit the root `Application` object itself.
15. Never hand-edit or ship manifests for `argocd-cm`, `argocd-rbac-cm`, `argocd-cmd-params-cm` or `argocd-secret`. They are operator-owned and reverted on reconcile. Change the `ArgoCD` CR instead; use `spec.extraConfig` only for `argocd-cm` keys the CRD genuinely doesn't expose.
16. `spec.sourceNamespaces` on the `ArgoCD` CR is an explicit literal list of team `-apps` namespaces. Never `"*"`, never a glob (`team-*`) or regex (`/^team-.*$/`) — the operator expands both at reconcile time, so a future namespace can silently join the allowlist.
17. Label discipline is part of the boundary: `argocd.argoproj.io/managed-by` goes only on workload namespaces; never on an `-apps` namespace; `argocd.argoproj.io/managed-by-cluster-argocd` is never written by hand. One namespace is managed by exactly one ArgoCD instance — infrastructure namespaces name `argocd-infra`, team namespaces name `argocd-multitenant`, and no namespace ever names both.
18. `ARGOCD_CLUSTER_CONFIG_NAMESPACES` on the operator's manager Deployment (set by `bootstrap/operator/manager-patch.yaml`) lists exactly `argocd-infra,argocd-multitenant` and nothing else. Every namespace added there gets an ArgoCD instance with cluster-wide controller permissions — it is a list of control planes, and a tenant namespace must never appear in it.
19. The `ArgoCD` CR **and its front door** (`install/argocd-gateway.yaml`) stay in `install/`, outside the path `platform-root` syncs, and are applied by a platform admin. ArgoCD never self-manages its own RBAC, SSO, instance config, or the gateway you would need to reach the UI. The control plane's gateway is a separate object from the tenant gateway, never a second listener on it.
20. The cluster runs Cilium, never Talos' default Flannel. The per-team `default-deny` policies are a core part of tenant isolation, and Flannel accepts them without enforcing them, so the failure mode is silent and total. Re-verify enforcement after any CNI change or cluster rebuild, not just after a policy change. Keep Cilium on a supported release — an EOL CNI is an EOL security boundary.
21. Every namespace this repo creates carries explicit `pod-security.kubernetes.io/*` labels, at `baseline` or stricter. Never rely on Talos' cluster default to supply the level, never add a namespace to the PSA exemption list in the Talos machine config (that disables admission for it entirely), and never widen a namespace's level without recording the reason in the manifest.
22. No OpenShift-only API surface, anywhere in this repo: no `Route`, no `spec.*.route` blocks on the `ArgoCD` CR, no `SecurityContextConstraints`, no `openshift-*` namespaces, no `oc` in any documented command. This is vanilla Kubernetes on Talos; most argocd-operator material found online is not.
23. Every team `AppProject` blacklists the whole `cilium.io` group (`kind: "*"`), not a list of named kinds. Cilium policy is additive, so a team-authored `CiliumNetworkPolicy` is a hole in the platform's `default-deny` that no `networking.k8s.io` blacklist entry catches — and a Cilium upgrade must never be able to open a new one by adding a CRD.
24. Every tenant gateway listener is per-team: a hostname wildcard under that team's own subdomain, `allowedRoutes.namespaces.selector` pinned to that team's workload namespace, and `kinds` limited to `HTTPRoute`/`GRPCRoute`. Never one shared listener with a broad selector, never `allowedRoutes.namespaces.from: All`. Gateway API breaks hostname conflicts by creation timestamp, so a shared listener makes hostname ownership a race between tenants.
25. Teams may author `HTTPRoute` and `GRPCRoute` and nothing else from `gateway.networking.k8s.io`; `Gateway`, `TLSRoute`, `TCPRoute`, `UDPRoute` and `ReferenceGrant` are blacklisted in every team `AppProject`. `Gateway` is namespaced — `clusterResourceWhitelist: []` does not cover it. Because this is an enumeration and not a wildcard, re-review it on every Gateway API upgrade: a new namespaced kind in the group is permitted until someone adds it.
26. `bootstrap/operator/rendered.yaml` is the only thing ever applied to the cluster for the operator, and it is always regenerated by `kustomize build`, never hand-edited — a hand-edit is erased by the next render and by the CI drift check. Equally, never `kubectl edit` the live operator `Deployment`, `ClusterRole` or CRDs: change the kustomization, re-render, re-apply.
27. The `infra` AppProject's `clusterResourceWhitelist` is the only non-empty one in this repo. It is safe only because the instance holding it has no tenants — no `sourceNamespaces`, no team group in its RBAC, its own hostname. Never copy its shape into a team project, and never grant a tenant access to the `argocd-infra` instance or the `infra` project. Changes to `infra/projects/infra-project.yaml` and `install/argocd-infra-cr.yaml` are one review, not two.
28. `argocd-infra` keeps `spec.sourceNamespaces` empty. Applications-in-any-namespace stays off for the infrastructure instance: every `Application` it reconciles lives in `argocd-infra`, a namespace only the platform team can write to.
29. No team SSO group ever appears in `argocd-infra`'s `spec.rbac.policy`, and the infrastructure instance is never given a team's repository, namespace or project. The two instances share the operator and nothing else.
30. step-ca's key material is created once, out of band, and supplied to the chart as pre-existing secrets with generation disabled. A chart-generated CA on a re-sync silently rotates the cluster's root of trust and invalidates every certificate issued from it. The material never enters Git (guardrail 12) — `infra/pki/step-ca-secrets.md` documents the procedure, never the keys.
31. The sealed-secrets controller's private sealing key is backed up out of band, before anything is sealed, and never committed. Losing it makes every `SealedSecret` in this repo and in all team repos undecryptable simultaneously; there is no recovery except regenerating every secret. Back up the key on cluster creation and after any key rotation.
32. `SealedSecret`s stay strict-scoped — bound to their namespace *and* name. Never seal with `--scope namespace-wide` or `--scope cluster-wide`. When the same plaintext is needed in two namespaces, seal it twice; strict scoping is what stops one tenant's sealed material from being unsealed under another tenant's namespace.
33. Keycloak configuration — realms, clients, groups, role mappings, protocol mappers — is authored as EDP operator CRs under `infra/identity/` and never changed in the Keycloak admin console. A console edit is drift the operator may or may not revert, and it is invisible in review. This includes the group-membership mapper that puts `groups` into the token: without it, both instances' RBAC silently denies everything.
