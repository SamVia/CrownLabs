# Comprehensive Setup Guide: Lightweight CrownLabs Instances & Liqo Offloading

This plan serves as a complete, end-to-end documentation guide for booting up two lightweight instances from scratch and fully setting up a Liqo multi-cluster environment to offload them.


## Step-by-Step Execution Plan

### 1. Create the Main Cluster
Create a fresh local KinD cluster for the main environment.
```bash
kind create cluster --name crownlabs
```

### 2. Install the CrownLabs CRDs
Navigate to the operators directory and install the CRDs into the main cluster.
```bash
cd /mnt/c/Users/DELL/Desktop/crownlabs/CrownLabs/operators
make install-local
```

### 3. Create Minimal CrownLabs Resources
Create and apply a `lightweight-resources.yaml` file to provision a minimal, GUI-less `Container` environment, along with its required `Workspace`, `Tenant`, and `Instance` definitions.

```yaml
apiVersion: crownlabs.polito.it/v1alpha2
kind: Tenant
metadata:
  name: test.user
  labels:
    crownlabs.polito.it/operator-selector: local
spec:
  firstName: Test
  lastName: User
  email: test.user@example.com
  workspaces:
    - name: lightweight
      role: user
---
apiVersion: crownlabs.polito.it/v1alpha1
kind: Workspace
metadata:
  name: lightweight
  labels:
    crownlabs.polito.it/operator-selector: local
spec:
  prettyName: Lightweight Workspace
  quota:
    cpu: "4"
    memory: "4Gi"
    instances: 5
---
apiVersion: v1
kind: Namespace
metadata:
  name: workspace-lightweight
---
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-test-user
  labels:
    crownlabs.polito.it/operator-selector: local
---
apiVersion: crownlabs.polito.it/v1alpha2
kind: Template
metadata:
  name: tiny-container
  namespace: workspace-lightweight
spec:
  nodeSelector: {}
  prettyName: Tiny Container
  description: A lightweight container
  environmentList:
    - name: tiny-env
      environmentType: Container
      image: alpine:latest
      guiEnabled: false
      persistent: false
      mountMyDriveVolume: false
      resources:
        cpu: 1
        memory: 100M
        reservedCPUPercentage: 10
  workspace.crownlabs.polito.it/WorkspaceRef:
    name: lightweight
---
apiVersion: crownlabs.polito.it/v1alpha2
kind: Instance
metadata:
  name: instance-1
  namespace: tenant-test-user
spec:
  template.crownlabs.polito.it/TemplateRef:
    name: tiny-container
    namespace: workspace-lightweight
  tenant.crownlabs.polito.it/TenantRef:
    name: test.user
---
apiVersion: crownlabs.polito.it/v1alpha2
kind: Instance
metadata:
  name: instance-2
  namespace: tenant-test-user
spec:
  template.crownlabs.polito.it/TemplateRef:
    name: tiny-container
    namespace: workspace-lightweight
  tenant.crownlabs.polito.it/TenantRef:
    name: test.user
```
**Apply the file:**
```bash
kubectl --context kind-crownlabs apply -f lightweight-resources.yaml
```

### 4. Create the Remote Cluster
Create a second cluster that will act as the target for Liqo offloading.
```bash
kind create cluster --name crownlabs-remote
```

### 5. Install Liqo on Both Clusters
Install the Liqo networking mesh on both clusters.
```bash
# Main cluster
liqoctl install kind --cluster-id crownlabs --context kind-crownlabs

# Remote cluster
liqoctl install kind --cluster-id crownlabs-remote --context kind-crownlabs-remote
```

### 6. Peer the Clusters
Establish the network tunnel between the two clusters.
```bash
liqoctl peer --context kind-crownlabs --remote-context kind-crownlabs-remote --gw-server-service-type NodePort
```

### 7. Apply the Liqo Offloading Policy
Create and apply a `liqo-offloading.yaml` policy on the main cluster to instruct Liqo to offload instances from the `tenant-test-user` namespace to the `crownlabs-remote` cluster.

```yaml
apiVersion: offloading.liqo.io/v1beta1
kind: NamespaceOffloading
metadata:
  name: offloading
  namespace: tenant-test-user
spec:
  namespaceMappingStrategy: DefaultName
  podOffloadingStrategy: LocalAndRemote
  clusterSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: "liqo.io/cluster-name"
            operator: In
            values:
              - crownlabs-remote
```
**Apply the file:**
```bash
kubectl --context kind-crownlabs apply -f liqo-offloading.yaml
```

### 8. Run the Instance Operator
Finally, start the Instance Operator on the main cluster. It will process the two instances, create the pods, and because of the Liqo policy, the pods will be scheduled on the remote cluster!

First, ensure your terminal is pointing back to the main cluster:
```bash
kubectl config use-context kind-crownlabs
```

Then, boot up the operator:
```bash
make run-instance DOMAIN="crownlabsfake.polito.it"
```

## Verification
1. Run the Instance Operator in your first terminal as described in Step 8.
2. In a second terminal, run `kubectl --context kind-crownlabs get pods -n tenant-test-user -o wide` to verify the pods are running.
3. Look closely at the `NODE` column in the output:
   - If offloading **failed** or is disabled, the pods will be scheduled on `crownlabs-control-plane` (the local cluster node).
   - If offloading is **working correctly**, the pods will be scheduled on the Liqo virtual node (typically named `liqo-crownlabs-remote`). This confirms that the pods have successfully been intercepted and pushed to the remote cluster!

## Advanced: Enforcing Local-Only Execution (Not Offloaded)

Even if a Namespace is offloaded by Liqo, you might have specific instances that you want to keep strictly on the local cluster (e.g., for latency or data compliance reasons). 

You can achieve this by using the `nodeSelector` field directly on your `Instance` specification. 

> [!IMPORTANT]
> In CrownLabs, an Instance can only define a `nodeSelector` if the underlying `Template` explicitly enables it! To allow this, you must add `nodeSelector: {}` (an empty selector) to the `Template`'s spec, which we have now added to the `tiny-container` template above.

If you want to create an `instance-3-local` that stays on the main cluster, you would simply add this to your YAML:
```yaml
---
apiVersion: crownlabs.polito.it/v1alpha2
kind: Instance
metadata:
  name: instance-3-local
  namespace: tenant-test-user
spec:
  nodeSelector:
    kubernetes.io/hostname: crownlabs-control-plane
  template.crownlabs.polito.it/TemplateRef:
    name: tiny-container
    namespace: workspace-lightweight
  tenant.crownlabs.polito.it/TenantRef:
    name: test.user
```
Because Kubernetes strictly enforces this `nodeSelector`, it will refuse to schedule the pod onto the Liqo virtual node, keeping the workload 100% local!

## Advanced: Targeting Specific Remote Clusters

If you have multiple remote clusters peered with Liqo (e.g., `crownlabs-remote-1` and `crownlabs-remote-2`) and you want to specifically select exactly *where* an Instance is spawned at creation, you have two approaches:

### Approach 1: Namespace-Level Targeting (Best for tenants)
If you want **all** instances for a specific user/tenant to go to a particular cluster, you modify the `NamespaceOffloading` policy's `clusterSelector`. 
```yaml
  clusterSelector:
    nodeSelectorTerms:
      - matchExpressions:
          - key: "liqo.io/remote-cluster-id"
            operator: In
            values:
              - crownlabs-remote-2 # <-- Specify the exact cluster here!
```

### Approach 2: Instance-Level Targeting (Granular)
If you want to choose the cluster on a **per-instance** basis at creation, you can use the exact same `nodeSelector` trick we used for local execution!

Instead of targeting `crownlabs-control-plane`, you simply target the specific Liqo virtual node that represents the cluster you want:
```yaml
spec:
  nodeSelector:
    kubernetes.io/hostname: crownlabs-remote-2 # <-- The name of the specific Liqo virtual node!
```
*(Note: As always, your underlying `Template` must have `nodeSelector: {}` enabled for this to work!)*

### Approach 3: The Hybrid Approach (Current Setup)
You can actually combine both of the methods above simultaneously, which is exactly what our current setup does!
1. **The Default:** We applied a `NamespaceOffloading` policy to `tenant-test-user` targeting `crownlabs-remote`. This means every instance created by this user will automatically spawn on the remote cluster by default (like `instance-1` and `instance-2`).
2. **The Local Exception:** We used the `nodeSelector` specifically on `instance-3-local` to override the namespace policy and strictly enforce that this specific pod stays on `crownlabs-control-plane`.
3. **The Explicit Remote:** We used the `nodeSelector` on `instance-4-remote-explicit` to target the remote virtual node by name, bypassing the "default" logic and ensuring it lands exactly where we want.
