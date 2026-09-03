# DSO202 Practical 02: Kubernetes Storage and StatefulSets

## 1. Purpose
This repository contains the configuration and documentation for DSO202 Practical 02. The purpose of this practical is to demonstrate the implementation of various Kubernetes storage abstractions, including:
*   **Static and Dynamic Provisioning** of PersistentVolumes.
*   **Reclaim Policies** (Retain vs. Delete).
*   **Access Modes** (ReadWriteOnce) and their impact on pod scheduling.
*   **Stateful Workloads** using StatefulSets to provide stable network identities and persistent storage for a PostgreSQL database.

## 2. Software and Image Versions
*   **Operating System:** Ubuntu 24.04 LTS
*   **Docker Desktop:** v4.34.2
*   **Kind:** v0.24.0
*   **Kubectl:** v1.31.0
*   **Kubernetes (Cluster):** v1.36.1
*   **PostgreSQL Image:** `postgres:16-alpine`
*   **Nginx Image:** `nginx:1.30-alpine` / `nginx:1.31-alpine` (for rollout tests)

---

## 3. Rebuild Sequence
Follow these steps to rebuild the entire practical environment from a clean machine:

### 3.1 Host Preparation
```bash
# Create the host directory for static storage
mkdir -p ~/dso202-p2-storage
```

### 3.2 Cluster and Namespace Setup
```bash
# Create the Kind cluster
kind create cluster --config cluster/kind-cluster.yaml

# Apply Namespace and Storage Configuration
kubectl apply -f manifests/00-namespace.yaml
kubectl config set-context --current --namespace=dso202-practical-02
kubectl apply -f manifests/01-quota-and-limits.yaml
kubectl apply -f manifests/02-storageclass-retain.yaml
```

### 3.3 Deploy Static and Dynamic Storage
```bash
# Stage 2: Static Provisioning
kubectl apply -f manifests/03-pv-static.yaml
kubectl apply -f manifests/04-pvc-static.yaml
kubectl apply -f manifests/05-pod-static-writer.yaml

# Stage 3: Dynamic Provisioning
kubectl apply -f manifests/06-pvc-dynamic.yaml
kubectl apply -f manifests/07-pod-dynamic-writer.yaml

# Stage 4: Shared Storage
kubectl apply -f manifests/08-deployment-shared-pvc.yaml
```

### 3.4 Deploy Stateful Applications
```bash
# Stage 5 & 6: StatefulSet (Nginx)
kubectl apply -f manifests/09-service-webnote.yaml
kubectl apply -f manifests/10-statefulset-webnote.yaml
kubectl apply -f manifests/11-pod-client.yaml

# Stage 7: PostgreSQL
kubectl apply -f manifests/12-secret-postgres.yaml
kubectl apply -f manifests/13-service-postgres.yaml
kubectl apply -f manifests/14-statefulset-postgres.yaml
```

---

## 4. Evidence Capture and SQL Dump
Gathering the final state and SQL dump as required by Stage 8:
```bash
mkdir -p evidence
kubectl exec postgres-0 -- pg_dump -U taskuser -d tasktracker > evidence/tasktracker-dump.sql
kubectl get all -o wide > evidence/final-state-all.txt
kubectl get pv,pvc,storageclass -o wide > evidence/final-state-storage.txt
```

---

## 5. Cleanup Sequence
To fully remove the practical environment and reclaim system resources:

### 5.1 Remove Kubernetes Objects
```bash
# Delete workloads and claims
kubectl delete -f manifests/14-statefulset-postgres.yaml
kubectl delete -f manifests/10-statefulset-webnote.yaml
kubectl delete -f manifests/11-pod-client.yaml
kubectl delete -f manifests/08-deployment-shared-pvc.yaml
kubectl delete -f manifests/07-pod-dynamic-writer.yaml
kubectl delete -f manifests/05-pod-static-writer.yaml
kubectl delete pvc --all

# Delete static PV
kubectl delete pv pv-web-static
```

### 5.2 Remove Cluster and Host Files
```bash
# Reset kubectl context
kubectl config set-context --current --namespace=default

# Delete Kind cluster
kind delete cluster --name dso202-p2

# Remove host storage directory
rm -rf ~/dso202-p2-storage
```

---

## 6. Repository Structure
```text
dso202-practical-02/
├── README.md                      # Rebuild and cleanup instructions
├── cluster/
│   └── kind-cluster.yaml          # Cluster configuration with host mount
├── manifests/                     # Kubernetes manifests (00-14)
├── evidence/                      # Screenshots and command outputs
└── report/
    └── practical-02-report.md     # Assessed report
```