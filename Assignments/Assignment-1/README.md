# DSO202 Assignment 1: Task Tracker Walkthrough
**Student Name:** Namgay Wangchuk  
**Namespace:** `dso202-assignment-01`  
**Image Registry:** `02230290namgay` (Docker Hub)

---

## 1. Required Technical Notes

### 1.1 Task 1: Architecture Note
Following the pattern in **Practical 1**, I used a multi-node `kind` cluster. The **kube-apiserver** serves as the gateway for manifests. The **kube-scheduler** assigns application pods to the **worker node** based on resource availability, while the **kubelet** on the worker node manages the container runtime (Docker). **Deployments** were used for stateless tiers (Frontend/Backend) to ensure self-healing, while a **PersistentVolumeClaim** was used for the database to ensure data persistence beyond the pod's lifecycle.

### 1.2 Task 2: Secret Security Note
**Note:** Kubernetes Secrets are base64-encoded, not encrypted at rest by default. They provide obfuscation within manifests but require additional encryption layers for true production-grade security.

### 1.3 Task 6: Governance Justification
The `ResourceQuota` limits memory to 1Gi. This is justified because the sum of the application's container requests (~600Mi total) allows all 4 pods (2 Backend, 1 Frontend, 1 DB) to operate with headroom for bursts while preventing any one tier from undergoing a memory leak that could exhaust cluster node resources.

---

## 2. Walkthrough & Manifest Files

### 2.1 Cluster Setup
The cluster is initialized with a multi-node configuration (Control-Plane + Worker) as per the Practical 1 guide.
```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  extraPortMappings:
  - containerPort: 30081
    hostPort: 30081
    protocol: TCP
```
**Command:** `kind create cluster --config kind-config.yaml`

### 2.2 Phase 1: Namespace & Governance
**File: `k8s/00-namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dso202-assignment-01
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-quota
  namespace: dso202-assignment-01
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
---
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-cpu-limits
  namespace: dso202-assignment-01
spec:
  limits:
  - default: { cpu: 500m, memory: 512Mi }
    defaultRequest: { cpu: 200m, memory: 256Mi }
    type: Container
```

### 2.3 Phase 2: Configuration & Secrets
**File: `k8s/01-config.yaml`**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dso202-assignment-01
data:
  DB_HOST: "db-service"
  DB_PORT: "5432"
  DB_NAME: "taskdb"
  APP_PORT: "8080"
  CORS_ORIGIN: "*"
  POSTGRES_DB: "taskdb"
  BACKEND_URL: "http://localhost:30080"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: dso202-assignment-01
type: Opaque
stringData:
  DB_USER: "taskuser"
  DB_PASSWORD: "taskpass"
  POSTGRES_USER: "taskuser"
  POSTGRES_PASSWORD: "taskpass"
```

### 2.4 Phase 3: Database Tier (Persistence & Headless Service)
**Files: `k8s/02-storage.yaml` & `k8s/03-db.yaml`**
```yaml
# 03-db.yaml snippet
apiVersion: v1
kind: Service
metadata:
  name: db-service
  namespace: dso202-assignment-01
spec:
  clusterIP: None # Headless Service
  selector: { tier: database }
  ports: [{ port: 5432 }]
```

### 2.5 Phase 4: Backend Tier (ClusterIP)
**File: `k8s/04-backend.yaml`**
```yaml
# 04-backend.yaml snippet
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: dso202-assignment-01
spec:
  type: ClusterIP # Internal Only
  selector: { tier: backend }
  ports: [{ port: 8080 }]
```

### 2.6 Phase 5: Frontend Tier (NodePort)
**File: `k8s/05-frontend.yaml`**
```yaml
# 05-frontend.yaml snippet
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: dso202-assignment-01
spec:
  type: NodePort
  selector: { tier: frontend }
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30081
```

---

## 3. Task 7: Verification Evidence (Transcripts)

**7b. DNS Resolution:**
```bash
kubectl exec -it <frontend-pod-name> -n dso202-assignment-01 -- curl -I http://backend-service:8080/api/tasks
# Output: HTTP/1.1 200 OK
```

**7c. Self-Healing:**
```bash
kubectl delete pod <backend-pod-name> -n dso202-assignment-01
# Observed: ReplicaSet immediately scheduled a replacement pod on kind-worker.
```

**7d. Declarative vs. Imperative Comparison:**
- **Declarative:** `kubectl apply -f k8s/` - Standard method for managing Desired State; allows for version control and auditability.
- **Imperative:** `kubectl run imperative-test --image=nginx:alpine -n dso202-assignment-01` - One-off command useful for debugging but lacks a permanent configuration record.

---

## 4. Final Project Structure
```text
assignment-1/
├── k8s/
│   ├── 00-namespace.yaml
│   ├── 01-config.yaml
│   ├── 02-storage.yaml
│   ├── 03-db.yaml
│   ├── 04-backend.yaml
│   └── 05-frontend.yaml
├── evidences/            (Screenshots 1-15)
├── report/
│   └── assignment-01-report.md
├── kind-config.yaml
└── README.md