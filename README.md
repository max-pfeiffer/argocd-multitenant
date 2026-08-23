# argocd-multitenant

GitOps source of truth for two ArgoCD instances (`argocd-infra` and `argocd-multitenant`) on a
vanilla Kubernetes cluster provisioned by Talos Linux. See [CLAUDE.md](CLAUDE.md) for the full
architecture, the bootstrap order, and the guardrails.

## DNS configuration (do this first)

**Every hostname below must resolve before you install anything.** The addresses are static: they are
pinned with `io.cilium/lb-ipam-ips` on each `Gateway` and handed out by the
`CiliumLoadBalancerIPPool` in [`infra/network/cilium-lb-ippool.yaml`](infra/network/cilium-lb-ippool.yaml)
(`192.168.20.245–249`), so the records can be created up front — there is nothing to wait for and no
address to look up after the fact.

Create the records in whatever serves the `.lan` zone (Pi-hole, dnsmasq, the router, an internal
BIND). All records are `A` records; no PTR records are required.

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
  `oidcConfig` and the hostname Keycloak itself is configured with (`spec.hostname.hostname`). A name
  that resolves differently for the browser and for the cluster breaks the OIDC redirect in a way
  that looks like an IdP misconfiguration.
- **Do not change an address without changing both places.** The pin lives on the `Gateway`
  annotation, and the pool's range in `infra/network/cilium-lb-ippool.yaml` has exactly five
  addresses. Adding a sixth endpoint means widening the pool first.
- **Onboarding a new team adds a DNS record.** A `*.team-<x>-amt.lan` wildcard pointed at the tenant
  gateway address belongs in the same reviewed PR as the listener, the certificate and the
  `NetworkPolicy` exception.

Verify what the cluster actually handed out once the gateways are up:

```bash
kubectl get gateway -A -o custom-columns=\
NS:.metadata.namespace,NAME:.metadata.name,ADDRESS:.status.addresses[*].value
```

Each address must match the table above; if one differs, DNS is pointing at the wrong service.
