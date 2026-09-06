# DSO202 Assignment 1: Task Tracker Walkthrough
**Student Name:** Namgay Wangchuk  
**Namespace:** `dso202-assignment-01`  
**Image Registry:** `02230290namgay` (Docker Hub)

---

## 1. Required Technical Notes

### 1.1 Task 1: Architecture Note
The **kube-apiserver** serves as the gateway for manifests. The **kube-scheduler** assigns pods to nodes based on **ResourceQuotas**, while the **kubelet** manages the container runtime (Docker) on the node. **Deployments** were used for stateless tiers (Frontend/Backend) to ensure self-healing via ReplicaSets, while a **PersistentVolumeClaim** was used for the database to ensure data persistence beyond the pod's lifecycle.

### 1.2 Task 2: Secret Security Note
**Note:** Kubernetes Secrets are base64-encoded, not encrypted at rest by default. They provide obfuscation within manifests but require additional encryption layers (such as KMS or Vault) for true production-grade security.

### 1.3 Task 6: Governance Justification
The `ResourceQuota` limits memory to 1Gi. This is justified because the sum of the application's container requests (~600Mi total) allows all 4 pods to operate with headroom for bursts while preventing any one tier from undergoing a memory leak that could exhaust cluster node resources.

---

## 2. Walkthrough & Manifest Files

### 2.1 Cluster Setup
The cluster is initialized using a Kind configuration that maps the Frontend NodePort to the host machine.
```bash
kind create cluster --config kind-config.yaml
```

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
# 02-storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: dso202-assignment-01
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
# 03-db.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-deployment
  namespace: dso202-assignment-01
  labels: { tier: database }
spec:
  replicas: 1
  selector:
    matchLabels: { tier: database }
  template:
    metadata:
      labels: { tier: database }
    spec:
      containers:
      - name: postgres
        image: 02230290namgay/dso202-db:1.0
        env:
        - name: POSTGRES_DB
          valueFrom: { configMapKeyRef: { name: app-config, key: POSTGRES_DB } }
        - name: POSTGRES_USER
          valueFrom: { secretKeyRef: { name: app-secret, key: POSTGRES_USER } }
        - name: POSTGRES_PASSWORD
          valueFrom: { secretKeyRef: { name: app-secret, key: POSTGRES_PASSWORD } }
        volumeMounts:
        - name: db-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: db-storage
        persistentVolumeClaim: { claimName: postgres-pvc }
---
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
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  namespace: dso202-assignment-01
  labels: { tier: backend }
spec:
  replicas: 2 # High Availability
  selector:
    matchLabels: { tier: backend }
  template:
    metadata:
      labels: { tier: backend }
    spec:
      containers:
      - name: backend
        image: 02230290namgay/dso202-backend:1.0
        env:
        - name: DB_HOST
          valueFrom: { configMapKeyRef: { name: app-config, key: DB_HOST } }
        - name: DB_NAME
          valueFrom: { configMapKeyRef: { name: app-config, key: DB_NAME } }
        - name: DB_USER
          valueFrom: { secretKeyRef: { name: app-secret, key: DB_USER } }
        - name: DB_PASSWORD
          valueFrom: { secretKeyRef: { name: app-secret, key: DB_PASSWORD } }
---
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
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  namespace: dso202-assignment-01
  labels: { tier: frontend }
spec:
  replicas: 1
  selector:
    matchLabels: { tier: frontend }
  template:
    metadata:
      labels: { tier: frontend }
    spec:
      containers:
      - name: frontend
        image: 02230290namgay/dso202-frontend:1.0
        env:
        - name: BACKEND_URL
          valueFrom: { configMapKeyRef: { name: app-config, key: BACKEND_URL } }
---
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

### 7b. DNS Resolution
Executed from the Frontend Pod to verify internal service discovery:
```bash
kubectl exec -it <frontend-pod-name> -n dso202-assignment-01 -- curl -I http://backend-service:8080/api/tasks
```
**Transcript Output:**
```text
HTTP/1.1 200 OK
X-Powered-By: Express
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
```

### 7c. Self-Healing
Manually deleted a backend pod and watched the ReplicaSet immediately schedule a replacement:
```bash
kubectl delete pod <backend-pod-name> -n dso202-assignment-01
# Observed via kubectl get pods -w:
# backend-deployment-xxx Terminating
# backend-deployment-yyy Pending -> Running
```

### 7d. Declarative vs. Imperative Comparison
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
```