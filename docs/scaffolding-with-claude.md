# Scaffolding this repo with Claude Code

`CLAUDE.md` is the specification: it describes what every file contains and why. This document is the **runbook** for
turning that specification into a working repo and a working cluster, using Claude Code to write the manifests.

Read `CLAUDE.md` end to end first. This file assumes you have, and does not repeat its reasoning — it only adds the
ordering, the decisions you have to make up front, and the verification gates between phases.

Each phase below is: a prompt you give Claude → files you review → a command **you** run against the cluster.

---

## Ground rules

**1. Claude writes manifests; you apply them.** Do not ask Claude to `kubectl apply` against this cluster. Every apply in
this repo is either a control-plane change or a tenant-boundary change, and guardrail 19 deliberately keeps this layer
outside any automatic reconciler. The point of that guardrail is a human in the loop — an agent running `kubectl apply` is
exactly the loop it exists to preserve.

**2. Never let Claude produce a digest, commit SHA, or version number.** A model will emit a syntactically perfect
`sha256:` string that points at nothing. Look these up yourself, paste them into the prompt, and instruct Claude to use
the literal value you supplied. Same for `spec.version` on the `ArgoCD` CR.

**3. Secrets go into Git only in sealed form** (guardrail 12). Every secret in this cluster is a `SealedSecret`; there is
no plaintext `Secret` in any manifest, and no value pasted into a Helm values block. Claude writes the `SealedSecret`
*shape* and you fill it by running `kubeseal` yourself — never paste plaintext into a prompt, because a prompt is a
transcript.

**4. One phase, one PR.** These diffs are security boundaries. A single "scaffold the whole repo" commit is not
reviewable, and the guardrails checklist below only works against small diffs.

**5. Re-read the guardrails a phase touches, after that phase.** They are numbered so you can cite them in review
comments.

**6. Every cluster command in this runbook uses the `admin@argocd-multitenant` context.** Set it once and re-check it at
the start of each phase that applies anything:

```bash
kubectl config use-context admin@argocd-multitenant
kubectl config current-context
```

Phases 3 through 9 install a cluster-wide operator, two ArgoCD control planes, CRDs and cluster RBAC. Landing those on the wrong cluster is not a
tidy mistake to undo — `--server-side --force-conflicts` will happily take ownership of a CRD belonging to some other Argo
CD. Check the context, do not assume it.

---

## Phase 0 — decisions, before you prompt anything

`CLAUDE.md` is written with placeholders. Resolve all of these first; a half-filled table is the main reason scaffolding
goes wrong, because Claude will happily leave `example.com` in place and it looks plausible in review.

| Value | Lands in | How to get it |
|---|---|---|
| kubectl context | every cluster command | fixed: **`admin@argocd-multitenant`** |
| Git URL of **this** repo | `platform/projects/platform-project.yaml`, `bootstrap/root-app.yaml` | your Git host |
| Git URL of each team's app repo | `platform/projects/<team>-project.yaml`, `platform/root-apps/<team>-root-app.yaml` | the teams; exact URLs, they are an allowlist (guardrail 5) |
| Team names | everywhere | default is `team-a`/`team-b`/`team-c` |
| ArgoCD hostname | CR `spec.server.host`, `install/argocd-gateway.yaml`, its certificate | you |
| Team hostname pattern | tenant gateway listeners | pick a per-team subdomain, e.g. `*.<team>.apps.example.com` (guardrail 24) |
| OIDC issuer URL + client ID | CR `spec.oidcConfig` | your IdP |
| IdP group names | CR `spec.rbac.policy` | your IdP — `platform-admins` plus one per team |
| cert-manager `ClusterIssuer` name | gateway certificates | your cluster |
| `StorageClass` name | only manifests that request a PVC | `kubectl get sc` |
| argocd-operator commit SHA | `bootstrap/operator/kustomization.yaml` `?ref=` | the operator repo's release tag → its commit SHA |
| argocd-operator image digest | same file, `images[].digest` | `crane digest quay.io/argoprojlabs/argocd-operator:vX.Y.Z` or your registry UI |
| Argo CD version | CR `spec.version` | the operator's compatibility matrix for the operator version above |
| Per-team quota sizing | `platform/namespaces/<team>.yaml` | capacity planning |
| **Infrastructure instance** | | |
| `argocd-infra` hostname | `install/argocd-infra-cr.yaml`, its gateway | you |
| LB address pool | `infra/network/cilium-lb-ippool.yaml` | fixed: **192.168.20.245–249**. Five addresses, four already spoken for — see the allocation table in CLAUDE.md |
| NFS server address + export path | `infra/apps/csi-driver-nfs.yaml`, the `StorageClass` | your storage |
| Keycloak hostname + realm name | `infra/apps/keycloak.yaml`; the `issuer` in **both** CRs' `oidcConfig` | you |
| step-ca root/intermediate key material | pre-existing secrets, created out of band | your secret manager — **never chart-generated**, guardrail 30 |
| Chart versions | each `infra/apps/*.yaml` `targetRevision` | given: cert-manager `v1.21.1`, step-certificates `1.30.1`. Pin the rest yourself |
| CloudNativePG chart version | `infra/apps/cloudnative-pg.yaml` | given: **v0.27.1** |
| Sealed Secrets version | `infra/apps/sealed-secrets.yaml` | pin a release from `github.com/bitnami/sealed-secrets` |
| Postgres cluster sizing + backup target | `infra/database/keycloak-postgres.yaml` | CNPG `Cluster` for Keycloak — name the `storageClass` explicitly |
| Keycloak realm groups | `infra/identity/groups.yaml` | `platform-admins` plus one `<team>-devs` per team, matching both CRs' `spec.rbac.policy` |

Open the prompt for Phase 1 with this table, and add: *"Use these exact values. Do not leave `example.com`, `<pin-this-…>`
or any other placeholder anywhere in the output."*

---

## Phase 1 — scaffold the file tree

Nothing touches the cluster in this phase.

```
Read CLAUDE.md, then create the full repo structure it describes: bootstrap/, install/, and platform/.

Use these values: <paste your Phase 0 table>

Rules:
- Follow CLAUDE.md exactly. Where it shows a manifest, use that manifest.
- Produce one file per path in the "Repo layout" section, and nothing else.
- team-b and team-c mirror team-a with names substituted — no other differences.
- Include infra/ in full: projects, network, storage, pki, and one Application per component
  with the sync-wave numbers from CLAUDE.md's ordering table.
- Leave the operator commit SHA and image digest as the literal values I gave you.
- Do not create bootstrap/operator/rendered.yaml — that is generated, not authored.
- Do not run kubectl.
```

Review before merging:

```bash
# Structure matches the doc
find bootstrap install platform -type f | sort

# Guardrail 5 — no wildcard sourceRepos/destinations in a TEAM project.
# Grep is the wrong tool here: the blacklist legitimately contains `kind: "*"` for cilium.io,
# so a naive search for '"*"' cries wolf. Check the two fields that matter, parsed:
python3 -c 'import yaml,glob,sys
for f in glob.glob("platform/projects/team-*.yaml"):
    d=yaml.safe_load(open(f))["spec"]
    bad=[r for r in d["sourceRepos"] if "*" in r]+[x for x in d["destinations"] if "*" in x["server"]+x["namespace"]]
    print(f, "FAIL" if bad else "ok", bad or "")'
# Guardrail 7 — the default project is empty
grep -A4 'name: default' platform/projects/default-project.yaml
# Guardrail 4 — every team Application disables namespace creation
grep -rL 'CreateNamespace=false' platform/root-apps/   # expect: no files listed
# Guardrail 15 — no operator-owned ConfigMaps shipped as manifests
grep -rn 'argocd-cm\|argocd-rbac-cm\|argocd-cmd-params-cm\|argocd-secret' platform/ install/ \
  | grep -v '^install/argocd-cr.yaml'                  # expect: no matches
# Guardrail 6 — each team project lists only its own -apps namespace
grep -A2 sourceNamespaces platform/projects/team-*.yaml
```

A useful second pass: `/code-review` on the diff, then read the guardrail list yourself. The automated pass catches
malformed YAML and copy-paste slips between teams; only you can catch a boundary that is subtly too wide.

---

## Phase 2 — cluster prerequisites (you, not Claude)

```bash
kubectl config use-context admin@argocd-multitenant   # first cluster contact — set it here
kubectl config current-context

kubectl -n kube-system exec ds/cilium -- cilium version
kubectl get crd tlsroutes.gateway.networking.k8s.io      # decides the ArgoCD front-door shape
kubectl get sc
kubectl -n kube-system get cm cilium-config -o jsonpath='{.data.enable-l2-announcements}'   # must be "true"
kubectl api-resources | grep -i -E 'ippool|l2announcement'   # confirms the CRD apiVersions to write
```

Do **not** install cert-manager here — the `argocd-infra` instance installs it in Phase 5. What you do need is
`l2announcements.enabled=true` on the Cilium release itself; the `CiliumL2AnnouncementPolicy` that Phase 5 applies is
silently inert without it, and the symptom is a `Gateway` that gets an address but never answers ARP.

Then check Talos' PSA defaults actually match what `CLAUDE.md` assumes, under `cluster.apiServer.admissionControl` in the
machine config.

**Feed the `TLSRoute` answer back to Claude** before Phase 6 — present means TLS passthrough, absent means the re-encrypt
fallback. Do not let it guess.

---

## Phase 3 — the operator

```
Fill in bootstrap/operator/kustomization.yaml with ref=<SHA> and digest=<DIGEST>, exactly as given.
Then walk me through the render/review/apply commands from CLAUDE.md — but do not run them.
```

You run:

```bash
kubectl config current-context        # must be admin@argocd-multitenant

kustomize build bootstrap/operator > bootstrap/operator/rendered.yaml
grep -A1 -E 'WATCH_NAMESPACE|ARGOCD_CLUSTER_CONFIG_NAMESPACES' bootstrap/operator/rendered.yaml
kubectl diff --server-side -f bootstrap/operator/rendered.yaml
kubectl apply --server-side --force-conflicts -f bootstrap/operator/rendered.yaml
kubectl wait --for=condition=Established crd/argocds.argoproj.io
kubectl -n argocd-operator rollout status deploy/argocd-operator-controller-manager
```

The `grep` is not optional. If the patch target name did not match upstream's generated Deployment name, the patch was a
silent no-op and you now have a namespace-scoped operator that will never reconcile your CR. Commit `rendered.yaml`.

---

## Phase 4 — the infrastructure ArgoCD instance

This instance goes up **before** the multi-tenant one, because it produces the storage class, the load-balancer address,
the certificate issuer and the identity provider that the multi-tenant instance needs.

```
Write install/argocd-infra-cr.yaml per CLAUDE.md's infrastructure instance section.
Leave spec.oidcConfig out entirely for now — Keycloak does not exist yet.
disableAdmin: false, sourceNamespaces: [], ingress disabled.
```

You run:

```bash
kubectl config current-context        # must be admin@argocd-multitenant

kubectl create namespace argocd-infra --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace argocd-infra \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted --overwrite

kubectl apply -f install/argocd-infra-cr.yaml
kubectl -n argocd-infra get pods
kubectl -n argocd-infra get secret argocd-infra-cluster -o jsonpath='{.data.admin\.password}' | base64 -d
kubectl -n argocd-infra port-forward svc/argocd-infra-server 8080:443
```

The port-forward is not a workaround, it is the only way in at this point: there is no LB address and no certificate yet,
and both are about to be installed by this instance. If Claude proposes `insecure: true` or a `NodePort` to "make it
reachable", decline — guardrail 11, and the port-forward disappears two phases from now.

---

## Phase 5 — the infrastructure applications, wave by wave

```
Write infra/projects/*.yaml and one Application per component under infra/apps/, using the
sync-wave table in CLAUDE.md. Pin every targetRevision to the versions in my Phase 0 table.
Also write infra/network/, infra/storage/ and infra/pki/ from CLAUDE.md.
Do not write any secret material — infra/pki/step-ca-secrets.md documents the procedure only.
```

You run — project first, then the root app, exactly as with `platform-root`:

```bash
kubectl apply -f infra/projects/infra-project.yaml
kubectl apply -f infra/projects/default-project.yaml
kubectl apply -f bootstrap/infra-root-app.yaml
kubectl -n argocd-infra get applications -w
```

**Two stops in this phase. Both are irreversible if you get them wrong.**

**Stop 1 — immediately after wave −2, back up the sealing key.** The sealed-secrets controller generates its private key
on first start. Everything you seal from here on — the CA material, the DB credentials, both OIDC client secrets, every
team's application secrets — is recoverable only with that key. Back it up before you seal anything (guardrail 31):

```bash
kubectl -n sealed-secrets get pods
kubectl -n sealed-secrets get secret -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml   > <your secret manager — NOT this repo, NOT /tmp>
```

**Stop 2 — before wave 3, make sure the sealed CA secrets are in Git and unsealed on the cluster.** If the
`step-certificates` chart syncs without finding existing CA secrets, it generates a fresh root CA. Notice that late and
every certificate already issued is untrusted, with a full PKI re-bootstrap as the fix. Generate the CA once, seal it,
commit it, and confirm the controller has unsealed it *before* wave 3 lands:

```bash
step certificate create "Platform Root CA" root.crt root.key --profile root-ca      # once, then destroy the plaintext
kubeseal --scope strict -n step-ca -o yaml < ca-secret.yaml > infra/pki/step-ca-sealedsecrets.yaml

kubectl -n step-ca get sealedsecret,secret     # both must exist before step-certificates syncs
kubectl -n argocd-infra get app step-certificates -o jsonpath='{.status.sync.status}'
```

Then verify each wave landed before caring about the next:

```bash
kubectl -n sealed-secrets get pods                         # wave -2
kubectl get ciliumloadbalancerippool default-pool          # wave -1
kubectl get sc                                             # wave 1 — exactly one default
kubectl -n cert-manager get pods                           # wave 2
kubectl get clusterissuer step-ca-acme                     # wave 4 — Ready=True
kubectl get clusters.postgresql.cnpg.io -A                 # wave 7 — Cluster in "Cluster in healthy state"
kubectl -n keycloak get keycloaks                          # wave 8
```

A wave that never goes healthy usually means a dependency from an earlier wave is missing rather than that the app is
broken — read the earlier waves first.

---

## Phase 6 — the infrastructure front door, and identity

Now there is an address and an issuer, so the port-forward can stop.

```bash
kubectl apply -f install/argocd-infra-gateway.yaml
kubectl -n argocd-infra get gateway argocd-infra          # expect 192.168.20.245
```

Point DNS at it and log in. Now configure Keycloak — all of it as EDP operator manifests under `infra/identity/`, none
of it in the admin console (guardrail 33):

```
Write infra/identity/: a KeycloakRealm, a KeycloakClient for argocd and one for argocd-infra,
a KeycloakRealmGroup for platform-admins and one per team (<team>-devs), and the
group-membership protocol mapper that puts a "groups" claim in the ID token.
Reference the client secrets by secretRef — I will seal them separately.
```

The OIDC client secret is needed in **two** namespaces: the EDP `KeycloakClient` reads it to configure Keycloak, and
Argo CD resolves `$argocd-oidc:clientSecret` from its own namespace. Seal the same plaintext twice rather than widening
the scope (guardrail 32):

```bash
kubeseal --scope strict -n keycloak          -o yaml < oidc-secret.yaml > infra/identity/client-secret-sealed.yaml
kubeseal --scope strict -n argocd-infra      -o yaml < oidc-secret.yaml > infra/identity/argocd-infra-oidc-sealed.yaml
kubeseal --scope strict -n argocd-multitenant -o yaml < oidc-secret.yaml > platform/argocd-oidc-sealed.yaml
```

Then add `spec.oidcConfig` to `install/argocd-infra-cr.yaml` and re-apply.

**Check the `groups` claim before declaring victory.** Login succeeding proves nothing — with `defaultPolicy: ""`, a token
without a `groups` claim authenticates fine and then fails every policy line, which looks exactly like broken RBAC:

```bash
argocd account get-user-info      # must list your groups, not just the username
```

Get SSO working on this instance before the multi-tenant one depends on the same realm.

---

## Phase 7 — the multi-tenant ArgoCD instance

Everything this instance assumed as a prerequisite now exists, courtesy of Phase 5: a default `StorageClass`, LB
addresses, and the `step-ca-acme` `ClusterIssuer` for its gateway certificate. Its `oidcConfig` points at the same
Keycloak realm as the infrastructure instance, with the `argocd` client rather than `argocd-infra`.

Ask Claude for the CR **with `spec.sourceNamespaces` left empty for now** — the team namespaces do not exist yet, and the
operator errors reconciling an entry it cannot `GET`. Also `disableAdmin: false` at this stage.

You run:

```bash
kubectl config current-context        # must be admin@argocd-multitenant

kubectl create namespace argocd-multitenant --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace argocd-multitenant \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted --overwrite

kubectl -n argocd-multitenant apply -f - <<EOF   # secret, by hand, never committed
apiVersion: v1
kind: Secret
metadata:
  name: argocd-oidc
  namespace: argocd-multitenant
  labels:
    app.kubernetes.io/part-of: argocd
stringData:
  clientSecret: <from your secret manager>
EOF

kubectl apply -f install/argocd-cr.yaml
kubectl -n argocd-multitenant get pods
kubectl -n argocd-multitenant get cm argocd-cm -o jsonpath='{.data.url}'   # must be your external URL
kubectl -n argocd-multitenant get secret argocd-cluster -o jsonpath='{.data.admin\.password}' | base64 -d
```

---

## Phase 8 — the multi-tenant front door

Apply `install/argocd-gateway.yaml`, point DNS at the Gateway's address, and log in with the admin password above. Do not
disable admin yet — you need it to debug the next phase.

---

## Phase 9 — the ordering that actually matters

This is the part that goes wrong if improvised. There is a genuine circularity at bootstrap: `platform-root` creates the
team namespaces, but the CR cannot list those namespaces until they exist, and the `platform` AppProject that
`platform-root` references lives inside the directory `platform-root` syncs. Do it in this order and there is no error
window:

```bash
kubectl config current-context        # must be admin@argocd-multitenant

# a. Namespaces first, by hand, once. platform-root adopts them later.
kubectl apply -f platform/namespaces/
kubectl get ns team-a team-a-apps team-b team-b-apps team-c team-c-apps

# b. NOW add the -apps namespaces to spec.sourceNamespaces and re-apply the CR.
kubectl apply -f install/argocd-cr.yaml
#    The operator labels each one. Verify — guardrail 17:
kubectl get ns team-a-apps -o jsonpath='{.metadata.labels}'   # managed-by-cluster-argocd, and NO managed-by
kubectl get ns team-a      -o jsonpath='{.metadata.labels}'   # managed-by, and NO managed-by-cluster-argocd

# c. The platform AppProject, by hand, once — it is inside the path the next step syncs.
kubectl apply -f platform/projects/platform-project.yaml

# d. Finally the root app. From here on, platform/ is GitOps-managed.
kubectl apply -f bootstrap/root-app.yaml
kubectl -n argocd-multitenant get app platform-root
```

If you reverse (a) and (b), the operator logs reconcile errors for the missing namespaces. If you run (d) before (b), the
team root `Application` objects get created but sit inert forever — ArgoCD will not reconcile an `Application` in a
namespace that is not in `sourceNamespaces`, and the symptom is silence, not an error you will notice.

---

## Phase 10 — one team, end to end, including the boundary test

Push a placeholder `Application` into team-a's repo under `app-of-apps/`, then:

```bash
kubectl -n team-a-apps get applications
kubectl -n team-a get all
```

Then **test the boundary deliberately** — this is the acceptance test for the entire design, and it is worth doing once
per environment:

```
Write me a test Application manifest that team-a could commit to their own repo,
which tries to use project: team-b. I want to confirm it fails.
```

Commit it to team-a's `app-of-apps/`, and confirm it produces a `ComparisonError` rather than deploying. A team-authored
manifest naming another team's project must fail on team-b's `sourceRepos`/`sourceNamespaces` not matching team-a's repo.
If it *succeeds*, stop and fix the projects before onboarding anyone. Delete the test manifest afterwards.

Repeat the same for a manifest targeting `namespace: team-b` and one setting `CreateNamespace=true`.

---

## Phase 11 — lock down

1. Confirm a `platform-admins` member can log in via SSO.
2. Set `spec.disableAdmin: true` in **both** `install/argocd-cr.yaml` and `install/argocd-infra-cr.yaml`, re-apply each
   (guardrail 10). It is easy to remember the tenant-facing one and forget the infrastructure one, which is the instance
   with the cluster-scoped whitelist.
3. Only now add each team's SSO group, and only after that team's wiring is verified per Phase 10.
4. Confirm the sealing-key backup from Phase 5 is where you think it is, and that it is not in any Git repo.
5. Branch protection and `CODEOWNERS` on every team repo's `app-of-apps/` directory.
6. Run the drift check, the two-instance overlap check, and the audit commands from `CLAUDE.md`'s "Common tasks" once,
   so you know what clean looks like.

---

## Prompts that reliably go wrong

| Prompt | Why it is a bad idea |
|---|---|
| "Just apply it for me" | Ground rule 1. Every apply here is a boundary change |
| "Look up / generate the image digest" | It will invent one that looks right |
| "The sync is failing, add a wildcard to the project" | The failure is usually the boundary working. Diagnose before widening — guardrails 5 and 6 |
| "Set the url in argocd-cm" | Operator-owned; reverted on reconcile (guardrail 15). It goes on the CR |
| "Scaffold everything and apply it in one go" | Unreviewable, and it will interleave the Phase 9 ordering wrongly |
| "Add the team's Role and RoleBinding for their -apps namespace" | The operator owns that RBAC now; hand-writing it fights the reconciler |
| "Give the team `applications, create` so they can self-serve" | Guardrail 2. Self-service is Git-only, by design |
| "Let the infra instance manage the team namespaces too, it already has cluster access" | Guardrails 17 and 29. One namespace, one instance — that is the wall |
| "Add the team's repo to the infra project so they can ship a CRD" | Guardrail 27. The `infra` project's cluster-scoped whitelist is safe only because no tenant can reach it |
| "Just let the step-certificates chart generate the CA, we can rotate later" | Guardrail 30. You cannot rotate a root of trust "later" without reissuing everything under it |
| "Seal it cluster-wide so we only need one copy" | Guardrail 32. That produces material any namespace can unseal — the opposite of what strict scoping is for |
| "Set the client secret in the Keycloak console, it's faster" | Guardrail 33. Console state is invisible in review and may be reverted by the operator |
| "Here is the client secret, put it in the manifest" | Ground rule 3. It is now in a transcript. Rotate it |

## When Claude and CLAUDE.md disagree

`CLAUDE.md` wins, and the disagreement is worth reading: it usually means either the doc has drifted from a new operator
version, or the model is pattern-matching on the far more common Helm-chart or OpenShift setups. Both are worth a moment
before you decide which to change. If the doc is genuinely wrong, fix the doc in its own PR — do not work around it in a
manifest, or the next person inherits an undocumented exception.
