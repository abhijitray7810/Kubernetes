# Kubernetes Deep Dive — Complete Study & Interview Guide

> Last Updated: 04/05/2026 | Covers: Labels, Annotations, Pods, ReplicaSets, Deployments, Storage, Access Control, Probes, Scaling

---

## Table of Contents

1. [Core Architecture Overview](#1-core-architecture-overview)
2. [Labels & Annotations](#2-labels--annotations)
3. [Pod Spec Deep Dive](#3-pod-spec-deep-dive)
4. [ReplicaSet — Deep Dive](#4-replicaset--deep-dive)
5. [Deployments — Deep Dive](#5-deployments--deep-dive)
6. [Persistent Volumes (PV) & PVCs](#6-persistent-volumes-pv--pvcs)
7. [Access Control — Private Images, Privileges, Tokens](#7-access-control--private-images-privileges-tokens)
8. [Probe Types](#8-probe-types)
9. [OOM Kill](#9-oom-kill)
10. [Scaling Strategies](#10-scaling-strategies)
11. [Rollout & Rollback](#11-rollout--rollback)
12. [Interview Q&A](#12-interview-qa)

---

## 1. Core Architecture Overview

```
Cluster
├── Control Plane (Master Node)
│   ├── API Server          ← kubectl talks here
│   ├── etcd                ← cluster state (key-value store)
│   ├── Scheduler           ← assigns pods to nodes
│   └── Controller Manager  ← runs controllers (ReplicaSet, Deployment, etc.)
│
└── Worker Nodes
    ├── kubelet             ← talks to API server, manages pods on node
    ├── kube-proxy          ← networking / iptables rules
    └── Container Runtime   ← (containerd / CRI-O)
```

### Key Hierarchy

```
Cluster
  └── Namespace
        └── Deployment
              └── ReplicaSet
                    └── Pod
                          └── Container(s)
```

**Why this matters in interviews:** Each level is independently observable and manageable. Deleting a Deployment deletes ReplicaSets and Pods. Deleting just a Pod lets the ReplicaSet recreate it.

---

## 2. Labels & Annotations

### What are Labels?

Labels are **key-value pairs** attached to Kubernetes objects (pods, nodes, services, etc.).  
They are used for **selection, grouping, and filtering**.

```yaml
metadata:
  labels:
    app: my-app
    env: production
    release: alpha
    tier: frontend
```

### Auto-labeling: Pod Name = Label Name Rule

> **Important:** If you do NOT manually specify labels, Kubernetes does NOT auto-create them.  
> However — when a **Deployment** creates pods, it auto-injects a label like:
> ```
> pod-template-hash: <hash>
> ```
> and uses `app: <deployment-name>` from the template spec.  
> The Pod name itself is: `<deployment-name>-<replicaset-hash>-<pod-hash>`

### Label vs Annotation — Key Difference

| Feature | Labels | Annotations |
|---------|--------|-------------|
| Purpose | Selection & grouping | Metadata, non-identifying info |
| Used by selectors? | ✅ Yes | ❌ No |
| Can be queried? | ✅ Yes (`-l`) | ❌ No |
| Size limit | Small | Large (build info, URLs, configs) |
| Example | `app: nginx` | `build-url: https://ci.example.com/123` |

### Label Use Cases (Interview Favorites)

1. **Service routing** — Service selects pods via labels
2. **Canary deployments** — `release: canary` vs `release: stable`
3. **Node affinity** — Schedule pods on nodes with label `disktype: ssd`
4. **Network policies** — Allow traffic only between pods with matching labels
5. **RBAC** — Restrict access to resources with specific labels
6. **Monitoring** — Prometheus scrapes pods with `prometheus.io/scrape: "true"`
7. **Cost allocation** — `team: backend`, `project: payments`

### Practical Label Commands

```bash
# Apply label to existing pod
kubectl label pod my-pod env=production

# Show pods WITH their labels
kubectl get pods --show-labels

# Filter pods by label
kubectl get pods -l release=alpha
kubectl get pods -l app=myapp,env=prod   # AND condition
kubectl get pods -l 'env in (prod, staging)'  # IN condition
kubectl get pods -l 'env notin (dev)'    # NOT IN

# Get pod by label (returns pod name)
kubectl get pods -l app=myapp -o name

# Remove a label
kubectl label pod my-pod env-

# Apply labels to all pods matching a selector
kubectl label pods -l app=myapp version=v2
```

### Simple Label Example — pod.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    app: nginx
    env: production
    release: alpha
    work: tutorial
    maintainer: "yourname@gmail.com"   # Annotation-style info in label
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
kubectl apply -f pod.yml
kubectl get pods --show-labels
```

### Annotations Deep Dive

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Updated image to v2.1"
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    build-url: "https://jenkins.example.com/job/myapp/42"
    git-commit: "abc123def456"
    owner: "platform-team@company.com"
```

**Why annotations?**
- Store build metadata, Git SHA, JIRA ticket IDs
- Configure Ingress controllers (nginx annotations)
- Enable Prometheus scraping
- Record rollout change causes

---

## 3. Pod Spec Deep Dive

### Complete Pod Spec File

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: full-pod-example
  namespace: default
  labels:
    app: myapp
    tier: backend
  annotations:
    maintainer: "team@company.com"

spec:
  # ── Scheduling ──────────────────────────────────
  nodeName: worker-node-1        # Pin to specific node (bypasses scheduler)
  nodeSelector:                  # Schedule on nodes with this label
    disktype: ssd

  # ── Init Containers ─────────────────────────────
  initContainers:
  - name: init-db-check
    image: busybox
    command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']

  # ── Main Containers ─────────────────────────────
  containers:
  - name: app-container
    image: myapp:v1.2.3
    imagePullPolicy: Always      # Always | IfNotPresent | Never

    ports:
    - containerPort: 8080
      protocol: TCP

    # Resource limits
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"

    # Environment variables
    env:
    - name: APP_ENV
      value: "production"
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

    # Volume mounts
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: data-volume
      mountPath: /data

    # Probes
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 15
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10

    startupProbe:
      httpGet:
        path: /startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10

    # Security
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]

  # ── Volumes ─────────────────────────────────────
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: data-volume
    persistentVolumeClaim:
      claimName: my-pvc

  # ── Pull Secrets ────────────────────────────────
  imagePullSecrets:
  - name: registry-credentials

  # ── Restart Policy ──────────────────────────────
  restartPolicy: Always          # Always | OnFailure | Never

  # ── Service Account ─────────────────────────────
  serviceAccountName: my-app-sa

  # ── DNS / Networking ────────────────────────────
  dnsPolicy: ClusterFirst
  hostNetwork: false
  terminationGracePeriodSeconds: 30
```

### Pod Phases

| Phase | Meaning |
|-------|---------|
| Pending | Accepted but not yet running (scheduling, image pull) |
| Running | At least one container running |
| Succeeded | All containers exited 0 |
| Failed | At least one container exited non-zero |
| Unknown | Node communication lost |

---

## 4. ReplicaSet — Deep Dive

### What is a ReplicaSet?

A **ReplicaSet** ensures that a specified number of **identical pod replicas** are running at all times.

```
ReplicaSet
  ├── selector (matchLabels)   ← "Which pods do I own?"
  ├── replicas: 3              ← "How many should exist?"
  └── template                 ← "What pod to create if count is short?"
```

### ReplicaSet YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp        # ← MUST match template labels
  template:
    metadata:
      labels:
        app: myapp      # ← MUST match selector
    spec:
      containers:
      - name: myapp
        image: nginx:1.21
        ports:
        - containerPort: 80
```

### Selector — What Is It?

The **selector** is how the ReplicaSet identifies which pods it "owns."  
It checks running pods for matching labels. If pod count < replicas → create more. If pod count > replicas → delete some.

```bash
# Check ReplicaSet
kubectl get replicaset
kubectl describe replicaset my-replicaset

# Scale
kubectl scale replicaset my-replicaset --replicas=5
```

### Ownership & ownerReferences

Every pod created by a ReplicaSet has an `ownerReferences` field in its metadata:

```yaml
ownerReferences:
- apiVersion: apps/v1
  kind: ReplicaSet
  name: my-replicaset
  uid: abc-123-def
  controller: true
  blockOwnerDeletion: true
```

**Interview tip:** This is how Kubernetes knows "this pod belongs to that ReplicaSet." When the RS is deleted, the garbage collector deletes owned pods via this reference.

### Idempotency Check — "Selector Check"

When a ReplicaSet controller runs:

```
1. List all pods matching selector labels
2. Count them → N
3. If N < replicas → create (replicas - N) pods
4. If N > replicas → delete (N - replicas) pods
5. If N == replicas → do nothing (idempotent!)
```

**Key:** If you manually create a pod with matching labels, the ReplicaSet ADOPTS it (does not create a new one).

### Homogeneous vs Heterogeneous Pods in ReplicaSet

| Type | Meaning |
|------|---------|
| Homogeneous | All pods are identical (same image, same config) — this is what ReplicaSet does |
| Heterogeneous | Different pods with different configs — NOT possible with a single ReplicaSet; use StatefulSet or individual Deployments |

### Ephemeral Containers (Kill a Pod to Test Resilience)

```bash
# Kill a pod — ReplicaSet immediately creates a replacement
kubectl delete pod my-replicaset-abc12

# Watch it respawn
kubectl get pods -w
```

This demonstrates the **self-healing** feature of ReplicaSets.

### Scale-Down Pod Deletion Algorithm (04/05/2026 — Current behavior)

When scaling down, the ReplicaSet controller chooses **which pods to delete** using this priority order:

1. **Pending / unschedulable pods** are deleted first
2. Pods with annotation `controller.kubernetes.io/pod-deletion-cost` — **lowest value deleted first**
3. Pods on nodes with **more replicas** are deleted before pods on nodes with fewer replicas
4. Pods that were created **more recently** are deleted before older pods

```bash
# Set deletion cost to protect important pods from scale-down
kubectl annotate pod my-pod controller.kubernetes.io/pod-deletion-cost=100
```

---

## 5. Deployments — Deep Dive

### What is a Deployment?

A **Deployment** is a higher-level abstraction over ReplicaSets.  
It manages rolling updates, rollbacks, and versioned history of ReplicaSets.

```
Deployment
  └── ReplicaSet (v1, running)
  └── ReplicaSet (v2, running) ← during rolling update
  └── ReplicaSet (v3, scaled to 0) ← old, kept for rollback
```

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  annotations:
    kubernetes.io/change-cause: "Initial deploy v1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Extra pods during update
      maxUnavailable: 1     # Pods that can be down during update
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0.0
```

### Update Strategies

#### 1. RollingUpdate (default)

```
Before: [v1] [v1] [v1]
Step 1: [v1] [v1] [v2]   ← create v2, kill one v1
Step 2: [v1] [v2] [v2]
Step 3: [v2] [v2] [v2]
```

- Zero downtime
- Both versions run simultaneously during update
- **Use case:** stateless web apps

#### 2. Recreate

```
Before: [v1] [v1] [v1]
Step 1: []   []   []     ← kill ALL v1
Step 2: [v2] [v2] [v2]  ← start ALL v2
```

- **Downtime window** exists
- **Use case:** apps that cannot run two versions simultaneously (DB schema migrations)

### Homogeneous vs Heterogeneous Deployments

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Homogeneous | All pods same version | Standard deployment |
| Heterogeneous (Canary) | Some pods v1, some v2 | A/B testing, gradual rollout |

**Canary with labels:**

```bash
# Deploy canary (10% traffic)
kubectl apply -f deployment-v2-canary.yaml  # replicas: 1

# Stable (90% traffic)
kubectl apply -f deployment-v1-stable.yaml  # replicas: 9

# Both have label: app=myapp (Service selects both)
```

### Scale In/Out

```bash
# Scale out
kubectl scale deployment my-deployment --replicas=10

# Scale in
kubectl scale deployment my-deployment --replicas=2

# Autoscale (HPA)
kubectl autoscale deployment my-deployment --min=2 --max=10 --cpu-percent=70
```

### Rollout & Rollback

```bash
# Check rollout status
kubectl rollout status deployment/my-deployment

# View rollout history
kubectl rollout history deployment/my-deployment

# View specific revision
kubectl rollout history deployment/my-deployment --revision=2

# Undo (rollback to previous version)
kubectl rollout undo deployment/my-deployment

# Rollback to specific revision
kubectl rollout undo deployment/my-deployment --to-revision=1

# Pause rollout (for canary testing)
kubectl rollout pause deployment/my-deployment

# Resume rollout
kubectl rollout resume deployment/my-deployment
```

### Change Cause (Why Annotations Matter Here)

```bash
# Annotate before/after update so history is readable
kubectl annotate deployment my-deployment kubernetes.io/change-cause="Updated to v2.1, fixed auth bug"

# Now rollout history shows:
# REVISION  CHANGE-CAUSE
# 1         Initial deploy v1.0.0
# 2         Updated to v2.1, fixed auth bug
```

---

## 6. Persistent Volumes (PV) & PVCs

### The Problem

> **If you delete a cluster (or pod/node crashes), your data is GONE** — unless stored in a Persistent Volume.

### PV Hierarchy

```
StorageClass  ← defines HOW storage is provisioned (EBS, NFS, etc.)
     └── PersistentVolume (PV)    ← actual storage resource
               └── PersistentVolumeClaim (PVC)  ← pod's request for storage
                         └── Pod uses the PVC via volumeMount
```

### PersistentVolume (PV)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce        # RWO: single node read/write
  # - ReadOnlyMany       # ROX: multiple nodes read-only
  # - ReadWriteMany      # RWX: multiple nodes read/write
  persistentVolumeReclaimPolicy: Retain  # Retain | Recycle | Delete
  storageClassName: standard
  hostPath:              # For local dev only
    path: /mnt/data
```

### PersistentVolumeClaim (PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

### Use PVC in Pod

```yaml
spec:
  volumes:
  - name: data-vol
    persistentVolumeClaim:
      claimName: my-pvc
  containers:
  - name: app
    volumeMounts:
    - name: data-vol
      mountPath: /data
```

### What Happens When You Delete a Cluster?

| Object Deleted | Data Survives? |
|---------------|---------------|
| Pod | ✅ Yes (PVC still exists) |
| ReplicaSet | ✅ Yes |
| Deployment | ✅ Yes |
| Namespace | ⚠️ Depends on PV reclaim policy |
| Cluster (entire) | ✅ Yes if using cloud PV (EBS/GCP Disk) with `Retain` policy |
| PVC deleted | ⚠️ Data gone if reclaim=Delete; retained if Retain |

---

## 7. Access Control — Private Images, Privileges, Tokens

### Accessing Private Registry Images

```bash
# Step 1: Create secret with registry credentials
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=me@example.com

# Step 2: Reference in pod spec
spec:
  imagePullSecrets:
  - name: regcred
  containers:
  - name: app
    image: registry.example.com/myapp:v1
```

### RBAC — Role-Based Access Control

```
ServiceAccount → bound to → Role/ClusterRole → grants → permissions
```

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default

---
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
subjects:
- kind: ServiceAccount
  name: my-app-sa
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Tokens

Kubernetes auto-mounts a ServiceAccount token at:
```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

```bash
# View the token
kubectl exec my-pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/token

# Disable auto-mount (security best practice)
spec:
  automountServiceAccountToken: false
```

### Pod Security — Privilege & Capabilities

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 3000
  fsGroup: 2000
  allowPrivilegeEscalation: false
  privileged: false               # NEVER true in production
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]                 # Drop all Linux capabilities
    add: ["NET_BIND_SERVICE"]     # Add back only what's needed
  seccompProfile:
    type: RuntimeDefault
```

---

## 8. Probe Types

### Why Probes?

Without probes, Kubernetes has no way to know if your app is actually healthy or ready.  
A container might be `Running` but the app inside could be crashed or still initializing.

### Three Probe Types

#### 1. Liveness Probe — "Is the app alive?"

- If it fails → **kill and restart the container**
- Use for: detecting app deadlocks, infinite loops, crashed state

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10   # Wait before first check
  periodSeconds: 15          # Check every 15s
  timeoutSeconds: 5          # Timeout per check
  failureThreshold: 3        # Fail 3 times → restart
  successThreshold: 1
```

#### 2. Readiness Probe — "Is the app ready to serve traffic?"

- If it fails → **remove pod from Service endpoints** (stop sending traffic)
- Container is NOT restarted
- Use for: warm-up time, temporary dependency failures

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3
```

#### 3. Startup Probe — "Is the app done starting up?"

- Runs FIRST before liveness/readiness kicks in
- If it fails after `failureThreshold × periodSeconds` → **kill container**
- Use for: slow-starting apps (Java, legacy systems)

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10          # Allows up to 5 minutes (30×10) to start
```

### Probe Check Methods

| Method | Config Example | Use Case |
|--------|---------------|----------|
| `httpGet` | `path: /health, port: 8080` | HTTP services |
| `tcpSocket` | `port: 5432` | TCP services (databases) |
| `exec` | `command: ["pg_isready"]` | Custom shell checks |
| `grpc` | `port: 50051` | gRPC services |

### Probe Decision Matrix

```
App starting up?  → startupProbe is running
                       ↓ passes
Is app alive?     → livenessProbe running every periodSeconds
Is app ready?     → readinessProbe running every periodSeconds
                       ↓ fails
                  → Removed from Service, not restarted
```

---

## 9. OOM Kill

### What is OOM Kill?

**Out Of Memory Kill** — when a container exceeds its memory **limit**, the Linux kernel kills the process with signal `SIGKILL`. Kubernetes then restarts the container (if restartPolicy=Always).

```bash
# Check if pod was OOM killed
kubectl describe pod my-pod | grep -i oom
kubectl describe pod my-pod | grep "OOMKilled"

# Check exit code (137 = OOM killed)
kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}'
```

### How to Prevent OOM Kill

```yaml
resources:
  requests:
    memory: "256Mi"   # Scheduler uses this to place pod
  limits:
    memory: "512Mi"   # OOM kill happens if exceeded
```

**Best Practice:** Always set both requests and limits. `requests ≤ limits`.

### Replica Restart After OOM

When a pod in a ReplicaSet gets OOM killed:
1. Container restarts inside the same pod (kubelet restarts it)
2. Pod stays on the same node
3. If pod keeps crashing → `CrashLoopBackOff` with exponential backoff

---

## 10. Scaling Strategies

### Types of Scaling

| Type | Controller | Trigger |
|------|-----------|---------|
| Manual Scale-Out | `kubectl scale` | Human operator |
| HPA (Horizontal Pod Autoscaler) | HPA controller | CPU, memory, custom metrics |
| VPA (Vertical Pod Autoscaler) | VPA controller | Right-sizes requests/limits |
| KEDA | KEDA controller | Event-driven (queue length, etc.) |
| Cluster Autoscaler | Cloud provider | Node-level scaling |

### HPA Example

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Business Use Cases for Scaling

| Scenario | Strategy |
|----------|---------|
| Flash sale / traffic spike | HPA with CPU metric |
| Message queue processing | KEDA with queue depth metric |
| ML inference (GPU) | VPA + manual scaling |
| Multi-region HA | Cluster federation + HPA |
| Cost optimization off-hours | Scheduled CronJob to scale down |

### Error Budget & Application Architecture

- **Error Budget** = `100% - SLO` (e.g., 99.9% SLO → 0.1% error budget = ~8.7 hrs/year downtime)
- Scaling helps maintain SLO by distributing load
- Multi-replica deployments prevent single-pod failures from breaching error budget

---

## 11. Rollout & Rollback

### Full Rollout Workflow

```bash
# 1. Update image (triggers new rollout)
kubectl set image deployment/my-deploy app=myapp:v2.0

# 2. Watch rollout progress
kubectl rollout status deployment/my-deploy

# 3. Check history
kubectl rollout history deployment/my-deploy
# REVISION  CHANGE-CAUSE
# 1         kubectl apply --filename=v1.yaml
# 2         Updated to v2.0

# 4. If v2 is bad → rollback
kubectl rollout undo deployment/my-deploy

# 5. Rollback to specific revision
kubectl rollout undo deployment/my-deploy --to-revision=1

# 6. Every revision = a ReplicaSet (old ones scaled to 0, kept for rollback)
kubectl get replicasets
```

### How kubectl rollout works internally

```
Deploy v2 triggered
  → New ReplicaSet created (v2-rs)
  → Old ReplicaSet (v1-rs) scaled down 1 by 1
  → v2-rs scaled up 1 by 1
  → maxSurge controls extra pods
  → maxUnavailable controls downtime
  
Rollback triggered
  → Old RS scaled back up
  → New RS scaled to 0
```

---

## 12. Interview Q&A

### Labels & Annotations

**Q: What is the difference between labels and annotations?**  
A: Labels are key-value pairs used for **selecting and grouping** Kubernetes objects — Services use labels to route traffic, ReplicaSets use them to own pods. Annotations are also key-value pairs but are **non-identifying metadata** — they can be large, are not queryable by selectors, and are used to store build info, monitoring configs, and tool-specific metadata.

**Q: If you don't specify labels, does Kubernetes auto-assign them?**  
A: Not exactly. If you create a bare pod without labels, it gets no labels. But a Deployment auto-injects `pod-template-hash` into every pod it creates, and the deployment template's labels are copied to pods. The pod NAME is separate from labels — `pod-name ≠ label`.

**Q: What are real-world use cases for labels?**  
A: (1) Service routing — `app: backend` selects target pods. (2) Canary deployments — `release: canary` subset. (3) Node affinity — `disktype: ssd` scheduling. (4) Network policies — allow traffic between pods with matching labels. (5) Cost tagging — `team: payments, project: checkout`.

**Q: What happens if you manually create a pod with labels matching a ReplicaSet selector?**  
A: The ReplicaSet adopts that pod. If the RS already has 3 desired replicas and you add a matching pod, it now sees 4 pods and will delete one (scale-down algorithm applies).

---

### Pods & ReplicaSets

**Q: What is ownerReferences in Kubernetes?**  
A: `ownerReferences` is a field in a resource's metadata that links it to its parent/owner. For example, a pod created by a ReplicaSet has an ownerReference pointing to that RS. When the RS is deleted, the garbage collector uses this to delete owned pods. It enables Kubernetes's **cascading deletion** feature.

**Q: How does the ReplicaSet controller decide which pod to delete during scale-down?**  
A: It uses a prioritized algorithm: (1) Pending/unschedulable pods first, (2) pods with lowest `pod-deletion-cost` annotation, (3) pods on nodes with more replicas of the same RS, (4) most recently created pods last.

**Q: Why use a Deployment instead of a ReplicaSet directly?**  
A: Deployments provide rolling updates, rollback history, pause/resume, and strategy control. ReplicaSets alone cannot do rolling updates — you'd have to manually manage two RSes. Deployments automate this entire lifecycle.

**Q: What is OOM Kill and how do you debug it?**  
A: OOM Kill happens when a container exceeds its memory limit — the kernel sends SIGKILL (exit code 137). Debug with `kubectl describe pod` looking for `OOMKilled` in status, and `kubectl get pod -o json` to check `lastState.terminated.exitCode`. Fix by increasing memory limits or profiling the app for memory leaks.

---

### Probes

**Q: What is the difference between liveness and readiness probes?**  
A: **Liveness** answers "is this container alive?" — failure causes a restart. **Readiness** answers "is this container ready to serve traffic?" — failure removes it from Service endpoints but does NOT restart it. Use readiness for warm-up; use liveness for deadlock detection.

**Q: When would you use a startup probe?**  
A: For slow-starting applications (Java apps, apps loading large datasets) where the normal `initialDelaySeconds` isn't enough. Startup probe prevents liveness from killing a container that's just taking a long time to start. It runs exclusively until it passes, then liveness/readiness take over.

---

### Storage

**Q: If a pod is deleted, is data lost?**  
A: Only if using ephemeral storage (emptyDir, hostPath without PV). If the pod uses a PVC backed by a PV, the data survives pod deletion and even node failure. The PVC binds to the same PV when the pod is recreated.

**Q: What is the difference between PV and PVC?**  
A: A **PersistentVolume (PV)** is a cluster-level storage resource (like an EBS volume). A **PersistentVolumeClaim (PVC)** is a user's request for storage. Pods reference PVCs, not PVs directly. This abstraction lets admins manage storage independently of app developers.

---

### Deployments & Rollouts

**Q: How does a rolling update work internally?**  
A: A new ReplicaSet is created for the new version. Kubernetes gradually scales up the new RS and scales down the old RS, one pod at a time (controlled by `maxSurge` and `maxUnavailable`). At the end, the old RS is scaled to 0 but kept (for rollback). `kubectl rollout undo` simply reverses the process.

**Q: What is `kubernetes.io/change-cause` annotation?**  
A: It's an annotation on a Deployment that gets recorded in rollout history. You set it to describe WHY a change was made. `kubectl rollout history` shows this as CHANGE-CAUSE, making it easy to audit deployments.

**Q: What is an Error Budget in the context of Kubernetes deployments?**  
A: An error budget is the allowed downtime/error rate derived from your SLO. For a 99.9% SLO, your error budget is 0.1% (~43 min/month). Deployment strategies like rolling updates and canary releases help you consume as little error budget as possible during releases. Running multiple replicas ensures pod failures don't breach the budget.

---

## Quick Reference Card

```bash
# Labels
kubectl get pods --show-labels
kubectl get pods -l app=myapp
kubectl label pod my-pod env=prod
kubectl label pod my-pod env-                    # remove label

# ReplicaSet
kubectl get replicaset
kubectl scale rs my-rs --replicas=5
kubectl describe rs my-rs

# Deployment
kubectl apply -f deployment.yml
kubectl set image deploy/my-deploy app=myapp:v2
kubectl rollout status deploy/my-deploy
kubectl rollout history deploy/my-deploy
kubectl rollout undo deploy/my-deploy
kubectl rollout undo deploy/my-deploy --to-revision=1

# Pods
kubectl get pods -o wide
kubectl describe pod my-pod
kubectl logs my-pod -c container-name
kubectl exec -it my-pod -- /bin/sh
kubectl top pod my-pod                           # resource usage

# Storage
kubectl get pv
kubectl get pvc
kubectl describe pvc my-pvc

# Secrets / Access
kubectl create secret docker-registry regcred ...
kubectl get serviceaccounts
```

---

*Study tip: Practice each concept hands-on with `minikube` or `kind`. Interview questions always come from real debugging scenarios.*
