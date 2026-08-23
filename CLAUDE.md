# CLAUDE.md — ArgoCD Multi-Tenant Platform Repo

This repo is the GitOps source of truth for a single ArgoCD instance that serves three teams (`team-a`, `team-b`, `team-c`)
on one **vanilla Kubernetes cluster provisioned by Talos Linux** — not OpenShift, see "Cluster assumptions" below. It is
managed by the platform team. Team application repos are separate and out of scope for this repo — this repo only contains
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
| A default `StorageClass` | **present** | nothing here consumes it yet; name the class explicitly in any manifest that does |
| An address for `Gateway` LoadBalancer services | **verify** | on bare-metal Talos there is no cloud LB — the `Gateway`'s service stays `Pending` unless Cilium LB-IPAM (`CiliumLoadBalancerIPPool`) or BGP is configured. Platform-owned, cluster-level |
| cert-manager, or externally provisioned certs | **to install** | issues the certificates the gateway listeners and `argocd-server` reference; guardrail 11 keeps `insecure: false`, so real certificates are not optional |

Client-side tooling for the bootstrap steps: `kubectl` (with `kustomize` built in, or a standalone `kustomize` binary).
No OLM, no `operator-sdk`, no Helm.

These are cluster-level prerequisites owned by the platform team. They are **not** managed by this repo and not synced by
ArgoCD: they all have to exist before ArgoCD does.

Both versions above are load-bearing enough to record and re-check on upgrade, but nothing in this file depends on a
specific patch release: `kubectl -n kube-system exec ds/cilium -- cilium version` and
`kubectl get crd gateways.gateway.networking.k8s.io -o jsonpath='{.metadata.annotations}'` are the ground truth.

## Architecture summary

- ArgoCD is installed and managed by the [argocd-operator](https://argocd-operator.readthedocs.io/en/latest/) (argoproj-labs), **not** by the `argo-helm` chart. The operator itself is installed cluster-wide from pinned upstream manifests via Kustomize — **no OLM** — and the ArgoCD instance is declared as a single `ArgoCD` custom resource.
- The `ArgoCD` CR is named `argocd` and lives in namespace **`argocd-multitenant`** (not the default `argocd`). That namespace is listed in the operator's `ARGOCD_CLUSTER_CONFIG_NAMESPACES` env var, which makes it a **cluster-scoped instance** — required for Applications-in-any-namespace, which the operator refuses to configure for a namespace-scoped instance.
- **The operator owns `argocd-cm`, `argocd-rbac-cm`, `argocd-cmd-params-cm` and `argocd-secret`.** Hand-edits to those ConfigMaps are reverted on the next reconcile. Every setting that used to be a Helm value or a ConfigMap patch is now a field on the `ArgoCD` CR (`spec.rbac.policy`, `spec.oidcConfig`, `spec.sourceNamespaces`, `spec.disableAdmin`, …), with `spec.extraConfig` as the escape hatch for `argocd-cm` keys the CRD does not expose.
- Auth is OIDC/SSO only, configured via `spec.oidcConfig` (no Dex, no Keycloak — `spec.sso` is left unset). Local admin is disabled via `spec.disableAdmin: true` after bootstrap. IdP groups `team-a-devs` / `team-b-devs` / `team-c-devs` map to per-team, project-scoped ArgoCD roles; `platform-admins` maps to `role:platform-admin`.
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
│   └── root-app.yaml               # app-of-apps: ArgoCD manages this repo's own /platform dir
├── install/
│   ├── argocd-cr.yaml              # the ArgoCD custom resource — the whole instance config
│   └── argocd-gateway.yaml         # Gateway + route for the ArgoCD UI itself — hand-applied (guardrail 19)
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

Upstream is a kubebuilder scaffold, so it applies its own `namePrefix` and the manager Deployment ends up as something
like `argocd-operator-controller-manager` with a container named `manager`. **Confirm the generated names before trusting
the patches** — `kustomize build bootstrap/operator | grep -E '^  name:|kind:'` — and fix the patch targets if upstream has
renamed anything. A strategic-merge patch whose name does not match is silently a no-op, which here would mean an operator
running with the wrong scope.

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
              value: argocd-multitenant        # ONLY this namespace — see guardrail 18
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
  version: v<pin-this-argocd-version>       # pin the Argo CD version explicitly, never float
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
    issuer: https://idp.example.com/
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
    host: argocd.example.com                # still set: the operator derives argocd-cm's `url` from it
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
kubectl -n argocd-multitenant get cm argocd-cm -o jsonpath='{.data.url}'   # expect https://argocd.example.com
```

If it is empty or the in-cluster service name, set it via `spec.extraConfig: {url: https://argocd.example.com}`. That is
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

## Exposing ArgoCD and team workloads (Gateway API)

Cilium implements Gateway API, so nothing here uses `Ingress`. There are **two** gateways on purpose, and the split is the
same one as everywhere else in this repo: the control plane's front door is hand-applied and never synced, while the
tenant front door is GitOps-managed.

| Gateway | Lives in | Managed by | Serves |
|---|---|---|---|
| `argocd` | `install/argocd-gateway.yaml`, namespace `argocd-multitenant` | platform admin, `kubectl apply` | the ArgoCD UI/API only |
| `tenant` | `platform/gateway/tenant-gateway.yaml`, namespace `tenant-gateway` | `platform-root` | team workloads, one listener per team |

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
      hostname: argocd.example.com
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
    - argocd.example.com
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
      hostname: "*.team-a.apps.example.com"    # team-a can only ever claim names under its own subdomain
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
`HTTPRoute` for `shop.apps.example.com` before team-a does simply takes the hostname, and if team-a already had it, a
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
    - checkout.team-a.apps.example.com  # must fall under the listener's wildcard
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
    - https://git.example.com/platform/argocd-multitenant.git   # exact match, this repo only
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
    repoURL: https://git.example.com/platform/argocd-multitenant.git
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

Chicken-and-egg note: `platform-root` references the `platform` AppProject that lives inside the directory it syncs. Apply
`platform/projects/platform-project.yaml` by hand once, immediately before the root app; from then on ArgoCD owns it.

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
`<team>/<team>-apps/*` pattern. If the team needs inbound traffic, the same PR adds their listener to
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

**Upgrade Argo CD:** bump `spec.version` in `install/argocd-cr.yaml` and re-apply. **Upgrade the operator:** bump the
`?ref=` commit SHA and the image digest in `bootstrap/operator/kustomization.yaml`, re-render `rendered.yaml`, review the
diff (RBAC and CRD changes especially), then `kubectl apply --server-side --force-conflicts`. Back up the `ArgoCD` CR
first — nothing gates a breaking CRD change for you now. Do the two upgrades separately, and check the operator's
compatibility matrix before either.

**Expose a team workload:** the team commits an `HTTPRoute` in their own repo; the platform side is a listener on the
tenant gateway (`platform/gateway/tenant-gateway.yaml`), a cert-manager `Certificate` for that team's wildcard hostname in
the `tenant-gateway` namespace, a DNS record, and the `NetworkPolicy` exception allowing gateway→pod ingress. All four are
platform-owned and belong in the same reviewed PR that onboards the team.

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
7. The built-in `default` AppProject stays locked to empty `sourceRepos`/`destinations`/`clusterResourceWhitelist`. Platform-owned apps use the dedicated `platform` project, not an exception to this rule.
8. `permitOnlyProjectScopedClusters: true` on every team `AppProject`.
9. Team repo credentials (`Repository`/`argocd-repo-creds` secrets) are always scoped to their owning `AppProject` via the `project` field — never registered project-wide.
10. `spec.disableAdmin` stays `true` once OIDC is verified. Only flip it back for break-glass recovery, and flip it off again afterward.
11. `spec.server.insecure` stays `false`; ArgoCD is always reached over TLS. This holds however it is exposed — if a routing setup is awkward to terminate correctly, fix the routing, never drop the server to plaintext.
12. Application-level secrets never get committed in plaintext anywhere in a team repo or this repo — they arrive via Sealed Secrets, SOPS, or an external secret manager. The OIDC client secret lives in its own labelled `Secret`, never in `argocd-secret` and never in Git.
13. Pin every version explicitly: the operator manifests via a `?ref=<commit-sha>` on the Kustomize base and its image via `sha256:` digest, and Argo CD via `spec.version` on the CR. Never a floating tag, never a branch ref, never `latest`. A moved tag must not be able to change what a re-render produces.
14. The per-team root `Application` (`platform/root-apps/<team>-root-app.yaml`) is platform-owned — teams commit `Application` manifests inside the path it watches, they never edit the root `Application` object itself.
15. Never hand-edit or ship manifests for `argocd-cm`, `argocd-rbac-cm`, `argocd-cmd-params-cm` or `argocd-secret`. They are operator-owned and reverted on reconcile. Change the `ArgoCD` CR instead; use `spec.extraConfig` only for `argocd-cm` keys the CRD genuinely doesn't expose.
16. `spec.sourceNamespaces` on the `ArgoCD` CR is an explicit literal list of team `-apps` namespaces. Never `"*"`, never a glob (`team-*`) or regex (`/^team-.*$/`) — the operator expands both at reconcile time, so a future namespace can silently join the allowlist.
17. Label discipline is part of the boundary: `argocd.argoproj.io/managed-by` goes only on workload namespaces; never on an `-apps` namespace; `argocd.argoproj.io/managed-by-cluster-argocd` is never written by hand. One namespace is managed by exactly one ArgoCD instance.
18. `ARGOCD_CLUSTER_CONFIG_NAMESPACES` on the operator's manager Deployment (set by `bootstrap/operator/manager-patch.yaml`) lists only `argocd-multitenant`. Every namespace added there gets an ArgoCD instance with cluster-wide controller permissions.
19. The `ArgoCD` CR **and its front door** (`install/argocd-gateway.yaml`) stay in `install/`, outside the path `platform-root` syncs, and are applied by a platform admin. ArgoCD never self-manages its own RBAC, SSO, instance config, or the gateway you would need to reach the UI. The control plane's gateway is a separate object from the tenant gateway, never a second listener on it.
20. The cluster runs Cilium, never Talos' default Flannel. The per-team `default-deny` policies are a core part of tenant isolation, and Flannel accepts them without enforcing them, so the failure mode is silent and total. Re-verify enforcement after any CNI change or cluster rebuild, not just after a policy change. Keep Cilium on a supported release — an EOL CNI is an EOL security boundary.
21. Every namespace this repo creates carries explicit `pod-security.kubernetes.io/*` labels, at `baseline` or stricter. Never rely on Talos' cluster default to supply the level, never add a namespace to the PSA exemption list in the Talos machine config (that disables admission for it entirely), and never widen a namespace's level without recording the reason in the manifest.
22. No OpenShift-only API surface, anywhere in this repo: no `Route`, no `spec.*.route` blocks on the `ArgoCD` CR, no `SecurityContextConstraints`, no `openshift-*` namespaces, no `oc` in any documented command. This is vanilla Kubernetes on Talos; most argocd-operator material found online is not.
23. Every team `AppProject` blacklists the whole `cilium.io` group (`kind: "*"`), not a list of named kinds. Cilium policy is additive, so a team-authored `CiliumNetworkPolicy` is a hole in the platform's `default-deny` that no `networking.k8s.io` blacklist entry catches — and a Cilium upgrade must never be able to open a new one by adding a CRD.
24. Every tenant gateway listener is per-team: a hostname wildcard under that team's own subdomain, `allowedRoutes.namespaces.selector` pinned to that team's workload namespace, and `kinds` limited to `HTTPRoute`/`GRPCRoute`. Never one shared listener with a broad selector, never `allowedRoutes.namespaces.from: All`. Gateway API breaks hostname conflicts by creation timestamp, so a shared listener makes hostname ownership a race between tenants.
25. Teams may author `HTTPRoute` and `GRPCRoute` and nothing else from `gateway.networking.k8s.io`; `Gateway`, `TLSRoute`, `TCPRoute`, `UDPRoute` and `ReferenceGrant` are blacklisted in every team `AppProject`. `Gateway` is namespaced — `clusterResourceWhitelist: []` does not cover it. Because this is an enumeration and not a wildcard, re-review it on every Gateway API upgrade: a new namespaced kind in the group is permitted until someone adds it.
26. `bootstrap/operator/rendered.yaml` is the only thing ever applied to the cluster for the operator, and it is always regenerated by `kustomize build`, never hand-edited — a hand-edit is erased by the next render and by the CI drift check. Equally, never `kubectl edit` the live operator `Deployment`, `ClusterRole` or CRDs: change the kustomization, re-render, re-apply.
