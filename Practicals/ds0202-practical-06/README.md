# DSO202 Practical 06: Kubernetes Package Management with Helm

**Course:** DSO202 — DevOps & Cloud Infrastructure  
**Author:** Namgay Wangchuk  
**Date:** October 1, 2026  
**Repository:** [ds0202-practical-06](https://github.com/namgaywangchuk/ds0202-practical-06)

---

## Executive Summary

This practical lab focuses on managing Kubernetes applications using **Helm 3**. In **Demo Stage 1**, we explored release management lifecycle operations, including chart repository setup, release installation, configuration updates, values merging, deployment failure simulation, automatic rollbacks, and OCI artifact management.

---

## Environment Setup & Prerequisites

- **Kubernetes Cluster:** Minikube / Local Cluster
- **Namespace:** `dso202-helm`
- **Helm Version:** Helm v3.x
- **Target Application:** `podinfo` (v6.15.0)

---

## Demo Stage 1: Helm Release Lifecycle Management

### Step 1: Repository Management & Namespace Creation

We added the official `podinfo` Helm repository, refreshed the local cache, and created a dedicated namespace for testing:

```bash
helm repo add podinfo [https://stefanprodan.github.io/podinfo](https://stefanprodan.github.io/podinfo)
helm repo update
kubectl create namespace dso202-helm
```
#### Verification:

```Bash
helm repo list
kubectl get ns dso202-helm
```

### Step 2: Release Installation
Installed the initial release named my-podinfo with custom parameters (replicas set to 2 and a custom message):

```Bash
helm install my-podinfo podinfo/podinfo \
  --version 6.15.0 \
  -n dso202-helm \
  --set replicaCount=2 \
  --set ui.message="Hello from DSO202"
```

#### Key Verification Commands:

```Bash
helm list -n dso202-helm
helm status my-podinfo -n dso202-helm
kubectl get pods,svc -n dso202-helm
```

### Step 3: Application Endpoint Verification
To test application traffic, port forwarding was configured:

```Bash
# Terminal 1: Port-Forward
kubectl -n dso202-helm port-forward deploy/my-podinfo 8088:9898

# Terminal 2: Test Endpoint
curl -s http://localhost:8088 | grep -E '"(version|message)"'
```

#### Output:

```JSON
"version": "6.15.0",
"message": "Hello from DSO202",
```

### Step 4: Secret-Based Release Storage Inspection
Helm stores release metadata as Kubernetes Secrets in the target namespace:

```Bash
kubectl get secrets -n dso202-helm --show-labels
```

Decoding and extracting the underlying release state JSON:

```Bash
kubectl get secret sh.helm.release.v1.my-podinfo.v1 -n dso202-helm \
  -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip \
  | jq '{name, namespace, version, status: .info.status, apply_method, values: .config}'
```

#### Output:

```JSON
{
  "name": "my-podinfo",
  "namespace": "dso202-helm",
  "version": 1,
  "status": "deployed",
  "apply_method": "ssa",
  "values": {
    "replicaCount": 2,
    "ui": {
      "message": "Hello from DSO202"
    }
  }
}
```

### Step 5: Helm Upgrade & Value Inheritance
Upgraded the release to Revision 2 by setting a custom UI color:

```Bash
helm upgrade my-podinfo podinfo/podinfo \
  --version 6.15.0 -n dso202-helm \
  --set ui.color="#2e7d32"
```

### Step 6: Values Merging (--reuse-values)
Restored original configurations while preserving the new UI color using the --reuse-values flag:

```Bash
helm upgrade my-podinfo podinfo/podinfo --version 6.15.0 -n dso202-helm \
  --reuse-values --set replicaCount=2 --set ui.message="Hello from DSO202"
```

Resulting User Values (helm get values my-podinfo -n dso202-helm):

```YAML
USER-SUPPLIED VALUES:
replicaCount: 2
ui:
  color: '#2e7d32'
  message: Hello from DSO202
```

### Step 7: Simulating Deployment Failure
Simulated an invalid image deployment with a short wait timeout to trigger an intentional failure:

```Bash
helm upgrade my-podinfo podinfo/podinfo --version 6.15.0 -n dso202-helm \
  --set image.tag="invalid-tag-999" \
  --wait --timeout 30s

Checking release history confirmed Revision 5 entered a failed state:

```Bash
helm history my-podinfo -n dso202-helm
```

### Step 8: Release Rollback
Executed a rollback to revert to Revision 1:

```Bash
helm rollback my-podinfo 1 -n dso202-helm
```

#### Verification:

```Bash
helm history my-podinfo -n dso202-helm
kubectl get pods -n dso202-helm
```

The release history confirmed a new Revision (Revision 6) was generated with the description `Rollback to 1`.

### Step 9: OCI Repository Management (Optional Experiment)
Tested installing directly from an OCI container registry (ghcr.io):

```Bash
helm show values oci://ghcr.io/stefanprodan/charts/podinfo --version 6.15.0 | head -n 3

helm install my-podinfo oci://ghcr.io/stefanprodan/charts/podinfo \
  --version 6.15.0 -n dso202-helm --set replicaCount=2
```

### Step 10: Stage Cleanup
Cleaned up all resources created in Stage 1:

```Bash
helm uninstall my-podinfo -n dso202-helm
kubectl delete ns dso202-helm
```

### Conclusion & Next Steps
Stage 1 successfully demonstrated Helm's core capabilities in managing application releases, tracking states in Kubernetes secrets, merging values safely, handling failed rollouts, and working with both standard and OCI-based chart registries.

Upcoming Stages:

- Stage 2: Custom Chart Creation (webapp)

- Stage 3: Template Writing & Go Helpers (_helpers.tpl)

- Stage 4: Multi-Environment Configurations (values-dev.yaml, values-prod.yaml)