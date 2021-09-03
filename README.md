# pawel-flux-poc

This is PoC flux setup for our customers. See https://github.com/giantswarm/giantswarm/issues/18738.

## Notes
* We need to be extra extra careful that kyverno policies are in force. Otherwise this whole things stops working and customers are able to do whatever they want.
* To protect ourselves (*but no the customer*) from above we can create SA (the same as `automation` in `flux-system`) and set flux pods to use that. That approach would also require updates to the kyverno policies.
* In our MC customers are not allowed to create namespaces directly. They need to create Organization CRs so the examples would need further adaptations be be used in real world.

## Steps
**NOTE:** For each `flux create` command you can use `--export` flag to see what manifests it creates.

### 1. Setup flux
After completing this section we should have flux running. It should be ready to be used by our customers without 

#### 1.1. Create a kind cluster
```bash
kind create cluster
```

#### 1.2. Install flux-app
Create namespace first:
```bash
kubectl --context="kind-kind" create ns "flux-system"
```

Clone the flux-app repository:
```bash
rm -rf /tmp/giantswarm-flux-app-repo
git clone https://github.com/giantswarm/flux-app /tmp/giantswarm-flux-app-repo
```

And install the app:
```bash
helm --kube-context="kind-kind" install -n "flux-system" "flux-app" \
	/tmp/giantswarm-flux-app-repo/helm/flux-app
```

#### 1.3. Install webhooks
Webhooks here enforce that for `Kustomization` and `HelmRelease`  *outside* `giantswarm` and `monitoring` namespace:
* `spec.serviceAccountName` is required - it has to be from the same namespace so customers applications have the same permissions as their Service Accounts
* `.spec.sourceRef.namespace` must be the same as `.metadata.namespace` - this means that sources (like git repository) from other namespaces can't be used

Create source:
```bash
flux --context="kind-kind" --namespace="flux-system" \
	create source git "pawel-flux-poc" \
	--url="https://github.com/giantswarm/pawel-flux-poc" \
	--branch="main"
```

Create kustomization from path `webhook` to deploy `kyverno` with policies:
```bash 
flux --context="kind-kind" --namespace="flux-system" \
	create kustomization "webhook" \
	--source="GitRepository/pawel-flux-poc" \
	--path="webhook" \
	--prune=true
```

### 2. Create universe
The "universe" here is a term for customer's app that can provision organizations. It should be created in the default namespace which our customers have access to.

Create source (in real world it should be a separate repo):
```bash
flux --context="kind-kind" --namespace="default" \
	create source git "pawel-flux-poc" \
	--url="https://github.com/giantswarm/pawel-flux-poc" \
	--branch="main"
```

Create (and fail) kustomization from path `universe` :
```bash
flux --context="kind-kind" --namespace="default" \
	create kustomization universe \
	--source="GitRepository/pawel-flux-poc" \
	--path="universe" \
	--prune=true
```

The output should look like this:
```
✚ generating Kustomization
► applying Kustomization
✗ admission webhook "validate.kyverno.svc" denied the request:

resource Kustomization/universe/universe was blocked due to the following policies

flux-multi-tenancy:
  serviceAccountName: 'validation error: .spec.serviceAccountName is required. Rule
    serviceAccountName failed at path /spec/serviceAccountName/'
```

This is because of the kyverno policies applied in 1.3.

Let's create `automation` Service Account in the default namespace as we have in our Management Clusters:
```bash
kubectl --context="kind-kind" --namespace="default" create serviceaccount "automation"
```

And try (and fail again) to create kustomization from path `universe` but with the `--service-account` flag set:
```bash 
flux --context="kind-kind" --namespace="default" \
	create kustomization universe \
	--source="GitRepository/pawel-flux-poc" \
	--path="universe" \
	--prune=true \
	--service-account="automation"
```

This should fail again but this time with a different error:
```
✚ generating Kustomization
► applying Kustomization
✔ Kustomization created
◎ waiting for Kustomization reconciliation
✗ apply failed: Error from server (Forbidden): error when retrieving current configuration of:
Resource: "/v1, Resource=namespaces", GroupVersionKind: "/v1, Kind=Namespace"
Name: "org-a", Namespace: ""
from server for: "44ecf883-47ef-4001-94bb-692a7bea30a5.yaml": namespaces "org-a" is forbidden: User "system:serviceaccount:default:automation" cannot get resource "namespaces" in API group "" in the namespace "org-a"
Error from server

... ELIDED ...
```

This is because `automation` SA doesn't have any permissions set yet. Let's add them:
```bash
kubectl --context="kind-kind" --namespace="default" create clusterrolebinding \
	"default:automation:cluster-admin" \
	--serviceaccount="default:automation" --clusterrole="cluster-admin"
```

The step above would not exist in our MC setup because `automation` SA is maintained by `rbac-operator`.

And try to create kustomization from path `universe` one more time:
```bash 
flux --context="kind-kind" --namespace="default" \
	create kustomization universe \
	--source="GitRepository/pawel-flux-poc" \
	--path="universe" \
	--prune=true \
	--service-account="automation"
```

### 3. Create tenant (organization admin) app
Create source (in real world it should be a separate repo):
```bash
flux --context="kind-kind" --namespace="org-a" \
	create source git "pawel-flux-poc" \
	--url="https://github.com/giantswarm/pawel-flux-poc" \
	--branch="main"
```

Try (and fail) creating app:
```bash
flux --context="kind-kind" --namespace="org-a" \
	create kustomization universe \
	--source="GitRepository/pawel-flux-poc" \
	--path="tenant/org-a" \
	--prune=true
```

This is again because `spec.serviceAccountName` is required but not set. Let's fix it and set the SA created with /universe/ which has /edit/ role bound:
```bash
flux --context="kind-kind" --namespace="org-a" \
	create kustomization universe \
	--source="GitRepository/pawel-flux-poc" \
	--path="tenant/org-a" \
	--prune=true \
	--service-account="created-from-universe"
```

Now it should fail with this error:
```
✚ generating Kustomization
► applying Kustomization
✔ Kustomization created
◎ waiting for Kustomization reconciliation
✗ apply failed: Error from server (Forbidden): error when retrieving current configuration of:
Resource: "/v1, Resource=configmaps", GroupVersionKind: "/v1, Kind=ConfigMap"
Name: "cm-from-tenant-org-a", Namespace: "org-b"
from server for: "2014d9e1-538e-43ef-ae56-08c15415916b.yaml": configmaps "cm-from-tenant-org-a" is forbidden: User "system:serviceaccount:org-a:created-from-universe" cannot get resource "configmaps" in API group "" in the namespace "org-b"
```

This is expected. This is because it's trying to create a ConfigMap in the `orb-b` namespace and `created-from-universe` is not allowed to do that. *This is up to customer how they set that*, but this is one possible scenario, e.g. allowing kong admins to only mess with /kong/ organization.

And you can check that the ConfigMap in `org-a` namespace is still created:
```bash
kubectl --context="kind-kind" --namespace="org-a" get configmap
```

The output should be similar to:
```
NAME                   DATA   AGE
cm-from-tenant-org-a   0      5m8s
cm-from-universe       0      20m
kube-root-ca.crt       1      20m
```
