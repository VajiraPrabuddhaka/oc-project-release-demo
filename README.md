# Project Release Lifecycle — `saas-product` demo

This walkthrough demonstrates OpenChoreo's **project release lifecycle** end to end,
using the [`saas-product`](./clusterprojecttype-saas-product.yaml) `ClusterProjectType`.

The release lifecycle extends OpenChoreo's familiar pattern from components and
resources up to the whole **project**:

```
Project + (Cluster)ProjectType  →  ProjectRelease  →  ProjectReleaseBinding  →  data plane
```

- **`ClusterProjectType`** — a platform-engineer-authored template for a *kind of
  project*. `saas-product` provisions each project an isolated, cost-governed,
  zero-trust cell namespace per environment.
- **`ProjectRelease`** — an **immutable snapshot** of the project's parameters +
  the resolved type, cut automatically whenever the project's spec drifts.
- **`ProjectReleaseBinding`** — pins a release to an environment, **owns the cell
  namespace**, and renders the type's resources into it. You create one per
  environment you want the project deployed to (see step 3).

By the end you'll have created one project (`storefront`), deployed it into three
governed cell namespaces (dev / staging / prod), then **promoted a change across
environments** — the core value of the feature.

---

## What `saas-product` gives every project

| Capability | Rendered resource(s) |
|---|---|
| Cost governance, per environment | `ResourceQuota` + `LimitRange` |
| Zero-trust egress baseline | `NetworkPolicy` default-deny + always-on cluster DNS allow |
| Approved egress allowlist | one `NetworkPolicy` **per declared dependency** (`forEach`) |
| Opt-in internet egress (dev) | `NetworkPolicy` gated by an `environmentConfig` (`includeWhen`) |
| Gateway-only ingress | `NetworkPolicy` gated on the environment's gateway (`includeWhen`) |
| Shared platform config | `ConfigMap` |
| Optional secret sync | `ExternalSecret` gated on the DataPlane secret store (`includeWhen`) |
| Governance + guardrails | team / cost-center / data-classification labels, enforced by CEL `validations` |

---

## Files in this directory

| File | Purpose |
|---|---|
| [`clusterprojecttype-saas-product.yaml`](./clusterprojecttype-saas-product.yaml) | The demo `ClusterProjectType` |
| `README.md` | This walkthrough |

The `Project` and `ProjectReleaseBinding` manifests are inline in the steps below
(`kubectl apply -f -`), so this directory stays self-contained.

---

## Prerequisites

- A running OpenChoreo control plane with a data plane registered, and the
  `getting-started` samples applied (this provides the **`default` DeploymentPipeline**
  with `development`, `staging`, and `production` environments).
- `kubectl` pointing at the OpenChoreo **control plane** context.
- **(only if you enable `externalSecrets` in step 2)** the External Secrets Operator
  installed on the data plane, serving `external-secrets.io/v1`, plus a
  `ClusterSecretStore`. Left disabled by default, so ESO is not required for the demo.

> **Data-plane RBAC note.** The `saas-product` type renders `ResourceQuota` and
> `LimitRange`. Some data-plane agent builds ship a `ClusterRole` that does not yet
> grant those. If a binding reports `ResourceApplyFailed` on `resource-quota`, grant
> them on the data-plane cluster:
> ```bash
> kubectl patch clusterrole cluster-agent-dataplane-openchoreo-data-plane --type=json \
>   -p '[{"op":"add","path":"/rules/-","value":{"apiGroups":[""],"resources":["resourcequotas","limitranges"],"verbs":["*"]}}]'
> ```

---

## Walkthrough

### 1. Register the project type (platform engineer)

```bash
kubectl apply -f clusterprojecttype-saas-product.yaml
kubectl get clusterprojecttype saas-product
```

This is the one-time platform step: teams can now create projects of kind
`saas-product`.

### 2. Create a project (developer)

The developer declares *what the product is* — its team, tier, data
classification, and its **approved external dependencies** — and nothing about
Kubernetes:

```bash
kubectl apply -f - <<'EOF'
apiVersion: openchoreo.dev/v1alpha1
kind: Project
metadata:
  name: storefront
  namespace: default
  annotations:
    openchoreo.dev/display-name: Storefront
spec:
  deploymentPipelineRef:
    name: default
  type:
    kind: ClusterProjectType
    name: saas-product
  parameters:
    team: commerce
    costCenter: "4200"
    tier: premium
    dataClassification: confidential
    observability:
      logLevel: info
    dependencies:
      - name: stripe-api
        cidr: 34.194.0.0/16
        port: 443
      - name: inventory-db
        cidr: 10.20.30.0/24
        port: 5432
EOF
```

Creating the Project immediately cuts an **immutable `ProjectRelease`** and records
it on the project — but **nothing is deployed yet**:

```bash
kubectl get projectreleases -n default | grep storefront
kubectl get project storefront -n default -o jsonpath='{.status.latestRelease}{"\n"}'
# -> {"hash":"...","name":"storefront-xxxxxxxx"}

kubectl get projectreleasebindings -n default | grep storefront   # (none yet)
```

> **New behavior — plan for this.** A project is only running in an environment
> where a `ProjectReleaseBinding` exists. Unlike the old flow, deploying a
> component now assumes its project has been deployed first. The controller does
> **not** create bindings — that's a client/experience-layer action (step 3).

### 3. Deploy the project to its environments

Deploying a project means creating a `ProjectReleaseBinding` per environment. The
**UI, CLI, and MCP** all offer a one-shot "deploy to all environments" for this — for
example the CLI:

```bash
# experience-layer equivalent (deploys / promotes per environment)
occ project deploy storefront --namespace default --to development
occ project deploy storefront --namespace default --to staging
occ project deploy storefront --namespace default --to production
```

The kubectl-native (GitOps) equivalent is to author the bindings directly. Leave
`spec.projectRelease` empty — the controller **seeds it with the latest release**
and renders the cell namespace:

```bash
for env in development staging production; do
kubectl apply -f - <<EOF
apiVersion: openchoreo.dev/v1alpha1
kind: ProjectReleaseBinding
metadata:
  name: storefront-$env
  namespace: default
spec:
  owner:
    projectName: storefront
  environment: $env
EOF
done

# each binding gets pinned to the latest release and converges to Ready=True
kubectl get projectreleasebindings -n default -w   # Ctrl-C once all three are READY=True
```

Expected:

```
NAME                     PROJECT      ENVIRONMENT   RELEASE               READY
storefront-development   storefront   development   storefront-xxxxxxxx   True
storefront-production    storefront   production    storefront-xxxxxxxx   True
storefront-staging       storefront   staging       storefront-xxxxxxxx   True
```

> **What to notice:** the developer never created a namespace, a quota, or a
> network policy — and never wrote per-environment YAML. One `Project` plus three
> bindings produced three governed cell namespaces.

### 4. Inspect a cell namespace

```bash
# the binding surfaces the cell namespace it owns
NS=$(kubectl get projectreleasebinding storefront-development -n default -o jsonpath='{.status.namespace}')
echo "cell namespace: $NS"

kubectl get resourcequota,limitrange,networkpolicy,configmap -n "$NS"
```

You'll see the quota, the limit range, the `platform-config` ConfigMap, the
default-deny + DNS policies, and — the highlight — **one egress policy per declared
dependency**, generated by `forEach`:

```
networkpolicy.networking.k8s.io/allow-egress-stripe-api
networkpolicy.networking.k8s.io/allow-egress-inventory-db
```

### 5. Per-environment posture (`environmentConfigs`)

The same release renders differently per environment. Open internet egress and a
bigger quota in **dev**, while **prod** stays locked to defaults:

```bash
kubectl patch projectreleasebinding storefront-development -n default --type=merge -p '
spec:
  environmentConfigs:
    cpuQuota: "8"
    memoryQuota: "16Gi"
    allowInternetEgress: true'

# dev now has the opt-in policy; prod does not
DEV=$(kubectl get projectreleasebinding storefront-development -n default -o jsonpath='{.status.namespace}')
PROD=$(kubectl get projectreleasebinding storefront-production  -n default -o jsonpath='{.status.namespace}')
kubectl get networkpolicy allow-internet-egress -n "$DEV"   # exists
kubectl get networkpolicy allow-internet-egress -n "$PROD"  # NotFound (includeWhen skipped it)
```

> **Guardrail in action:** set the project's `dataClassification: restricted` and try
> `allowInternetEgress: true`, and the render is *rejected* — a CEL `validation`
> forbids that combination. Check `kubectl describe projectreleasebinding …` for the
> message.

### 6. Promote a change across environments

This is the payoff. Change the product — say, add a new approved dependency — and a
**new immutable release** is cut. Existing bindings stay pinned to the old release
(promotion is explicit), so you roll the change out environment by environment.

```bash
# 1) edit the project — add a dependency
kubectl patch project storefront -n default --type=json -p '[{
  "op":"add","path":"/spec/parameters/dependencies/-",
  "value":{"name":"payments-gw","cidr":"52.10.0.0/16","port":443}}]'

# 2) a NEW release is cut; latestRelease advances, bindings do NOT
kubectl get project storefront -n default -o jsonpath='{.status.latestRelease.name}{"\n"}'
kubectl get projectreleasebindings -n default          # RELEASE column still shows the old release

# 3) promote dev -> verify -> staging -> prod
NEW=$(kubectl get project storefront -n default -o jsonpath='{.status.latestRelease.name}')
kubectl patch projectreleasebinding storefront-development -n default --type=merge -p "{\"spec\":{\"projectRelease\":\"$NEW\"}}"
# ...confirm dev looks good, then:
kubectl patch projectreleasebinding storefront-staging     -n default --type=merge -p "{\"spec\":{\"projectRelease\":\"$NEW\"}}"
kubectl patch projectreleasebinding storefront-production  -n default --type=merge -p "{\"spec\":{\"projectRelease\":\"$NEW\"}}"

# the new allow-egress-payments-gw policy now exists in each promoted cell
kubectl get networkpolicy -n "$DEV" | grep payments-gw
```

> The same promotion is available from the **UI, CLI (`occ project deploy … --to <env>`),
> and MCP** — all just advance `spec.projectRelease`.

---

## Note: doing this from the Backstage UI

The steps above use `kubectl` to make the mechanics explicit, but most developers
drive the whole lifecycle from OpenChoreo's Backstage-powered **developer portal** —
which creates the exact same `Project` and `ProjectReleaseBinding` resources.

> Portal URL (k3d single-cluster): **http://openchoreo.localhost:8080** — the
> "OpenChoreo Console". Adjust for your install.

- **Create the project.** Create a new Project in the portal and pick its type
  (`saas-product`). The portal generates a form from the type's `parameters` schema
  — team, tier, data classification, dependencies — so developers fill in fields
  instead of writing YAML. Saving creates the `Project` and cuts its first
  `ProjectRelease` (equivalent to step 2).
- **Deploy it.** From the Project's **Deploy** tab, deploy the project to its
  environments. This is what creates the `ProjectReleaseBinding`s (step 3): the
  portal's "deploy to all environments" option seeds one binding per environment for
  you, so there is no separate manual step.
- **Promote.** Promotion runs from the same **Deploy** tab, at the **project** level
  — advancing an environment to a newer `ProjectRelease` (equivalent to step 6).
  Individual per-component promotion is not offered; promotions flow through the
  project, which is the whole point of the project release lifecycle.
- **Observe.** The portal shows each binding's status and the rendered data-plane
  resources — the same ones you inspected with `kubectl` in step 4.

Because the UI, CLI (`occ`), MCP, and GitOps all produce the same resources, you can
mix approaches — e.g. create and deploy from the portal, then inspect with `kubectl`.

---

## Key ideas this demo shows

- **One project → many governed namespaces.** Developers describe the product;
  the platform's type materializes isolation, cost, and network posture per env.
- **Projects are deployed explicitly.** Creating a project cuts a release but
  deploys nothing; a binding per environment (via UI / CLI / MCP / GitOps) is what
  puts it into a cell — the controller then pins unpinned bindings to the latest release.
- **Immutable releases + explicit promotion.** A release is a frozen snapshot;
  changes cut a new one, and you choose when each environment adopts it.
- **Per-environment configuration** from a single release via `environmentConfigs`.
- **Zero-trust by construction** — default-deny plus an explicit, `forEach`-generated
  egress allowlist derived from the product's declared dependencies.
- **Guardrails as code** — CEL `validations` block non-compliant combinations
  (e.g. restricted data + open internet egress) before anything is applied.

---

## Optional: enable secret sync

The type includes a shared-cell `ExternalSecret`, rendered only when the project
opts in **and** the DataPlane exposes a secret store. To try it, add to the
project's parameters:

```yaml
    externalSecrets:
      enabled: true
      keys: [db-password, stripe-key]
```

Requires the External Secrets Operator (serving `external-secrets.io/v1`) and a
`ClusterSecretStore` on the data plane. The `ExternalSecret` will apply; it only
turns *Ready* once the referenced keys (`storefront/<env>/<key>`) actually exist in
the backing store.

---

## Cleanup

```bash
# remove the bindings, then the project
kubectl delete projectreleasebinding -n default \
  storefront-development storefront-staging storefront-production
kubectl delete project storefront -n default

# ProjectReleases are immutable snapshots and are NOT garbage-collected with the
# project — delete them explicitly for a clean slate:
kubectl get projectreleases -n default | grep storefront
kubectl delete projectrelease -n default storefront-<hash>   # repeat per listed release

# remove the type (only if no other project uses it)
kubectl delete clusterprojecttype saas-product
```
