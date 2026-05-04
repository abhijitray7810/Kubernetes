# ☸️ Kubernetes Networking Deep Dive — Complete Guide

> **Covers:** Deployments → Services (ClusterIP, NodePort, LoadBalancer, Headless, ExternalName) → Endpoints → EndpointSlices → Ingress → WAF → Load Balancer Topology → Interview Q&A

---

## Table of Contents

1. [Deployment Basics](#1-deployment-basics)
2. [Services — Overview & Types](#2-services--overview--types)
3. [ClusterIP](#3-clusterip)
4. [NodePort](#4-nodeport)
5. [LoadBalancer](#5-loadbalancer)
6. [Headless Service](#6-headless-service)
7. [ExternalName](#7-externalname)
8. [Services With No Selectors](#8-services-with-no-selectors)
9. [Endpoints vs EndpointSlices](#9-endpoints-vs-endpointslices)
10. [Internal Pod vs External Pod Communication](#10-internal-pod-vs-external-pod-communication)
11. [IPs — Internal, External, Cloud Rules](#11-ips--internal-external-cloud-rules)
12. [WebSocket & LoadBalancer Considerations](#12-websocket--loadbalancer-considerations)
13. [Load Balancer Internal Topology Algorithms](#13-load-balancer-internal-topology-algorithms)
14. [Ingress — Full Deep Dive](#14-ingress--full-deep-dive)
15. [Ingress Controller vs Ingress Object](#15-ingress-controller-vs-ingress-object)
16. [Path-Based, Domain-Based, TLS Ingress](#16-path-based-domain-based-tls-ingress)
17. [DNS — A Record, CNAME, IngressClassName](#17-dns--a-record-cname-ingressclassname)
18. [WAF (Web Application Firewall) with Ingress](#18-waf-web-application-firewall-with-ingress)
19. [Full Traffic Flow Diagram](#19-full-traffic-flow-diagram)
20. [Interview Questions & Answers](#20-interview-questions--answers)

---

## 1. Deployment Basics

### Definition
A **Deployment** is a Kubernetes object that manages a ReplicaSet, which in turn manages Pods. It ensures the desired number of Pod replicas are always running.

### Why Needed
- Self-healing (restarts failed pods)
- Rolling updates with zero downtime
- Rollback support
- Scaling (manual or HPA)

### Use Cases
- Stateless web applications
- API servers
- Microservices

### Full YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app                          # Name of this deployment
  namespace: default                    # K8s namespace
  labels:
    app: my-app                         # Label used by Service selector
spec:
  replicas: 3                           # Desired number of pod copies
  selector:
    matchLabels:
      app: my-app                       # Must match template labels
  strategy:
    type: RollingUpdate                 # Zero-downtime update strategy
    rollingUpdate:
      maxSurge: 1                       # Max extra pods during update
      maxUnavailable: 0                 # No pod goes down during update
  template:
    metadata:
      labels:
        app: my-app                     # Pod label (matched by selector)
    spec:
      containers:
      - name: my-app
        image: nginx:1.25
        ports:
        - containerPort: 80             # Port container listens on
        resources:
          requests:
            cpu: "100m"                 # Minimum CPU guaranteed
            memory: "128Mi"
          limits:
            cpu: "500m"                 # Maximum CPU allowed
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5        # Wait before first probe
          periodSeconds: 10             # Check every 10s
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
```

### Line-by-Line Analysis
| Field | Purpose |
|-------|---------|
| `apiVersion: apps/v1` | API group for Deployment |
| `replicas: 3` | 3 pods always running |
| `selector.matchLabels` | How Deployment finds its Pods |
| `strategy.RollingUpdate` | Replace pods one by one, no downtime |
| `containerPort: 80` | Informational only; actual binding is in app |
| `readinessProbe` | Traffic only sent when pod is ready |
| `livenessProbe` | Restarts container if unhealthy |

---

## 2. Services — Overview & Types

### Definition
A **Service** is a stable virtual IP (ClusterIP) + DNS name that load-balances traffic across a set of Pods matched by label selector.

### Why Needed
Pods are ephemeral — they die and get new IPs. Services provide a stable endpoint.

### Types Summary

| Type | Accessible From | Use Case |
|------|----------------|----------|
| ClusterIP | Inside cluster only | Internal microservice communication |
| NodePort | Outside via `<NodeIP>:<NodePort>` | Dev/test, simple external access |
| LoadBalancer | Outside via cloud LB | Production external traffic |
| Headless | Direct pod DNS | StatefulSets, Kafka, Cassandra |
| ExternalName | CNAME redirect | External DB, 3rd-party APIs |

### Port Terminology
```
[Client] → NodePort (30000-32767 on Node)
              ↓
         Service Port (e.g., 80)          ← "port" in Service spec
              ↓
         Target Port (e.g., 8080)         ← "targetPort" in Service spec
              ↓
         Container Port (containerPort)   ← port app listens on
```

> **Default Rule:** If `targetPort` is not specified, it defaults to the same value as `port`.

---

## 3. ClusterIP

### Definition
Default service type. Creates a virtual IP accessible **only inside** the cluster.

### Why Needed
Internal communication between microservices without exposing to outside world.

### Use Cases
- frontend-svc → backend-svc
- app → database-svc
- any inter-pod communication

### YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: default
spec:
  type: ClusterIP                       # Default type, internal only
  selector:
    app: backend                        # Routes to pods with this label
  ports:
  - name: http
    protocol: TCP
    port: 80                            # Service listens on this port
    targetPort: 8080                    # Forwards to container port 8080
```

### How It Works
- kube-proxy programs iptables/IPVS rules on every node
- Any pod calling `http://backend-svc:80` → gets routed to one of the backend pods on port 8080
- DNS: `backend-svc.default.svc.cluster.local`

---

## 4. NodePort

### Definition
Exposes service on a **static port on every Node** in the cluster (range: 30000–32767).

### Why Needed
Simple external access without a cloud load balancer.

### Use Cases
- Development/testing
- On-premises clusters without LB support
- Exposing services to VPN-connected clients

### YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - name: http
    protocol: TCP
    port: 80                            # ClusterIP port (internal)
    targetPort: 3000                    # Container port
    nodePort: 30080                     # External port on every Node (optional, auto-assigned if omitted)
```

### Port Relationships
```
External User
    ↓
Node IP:30080   (nodePort — fixed on every node)
    ↓
Service:80      (port — ClusterIP level)
    ↓
Pod:3000        (targetPort — container)
```

### Problems with NodePort in Production
- Exposes node IPs directly (security risk)
- Not a real load balancer (no health checks on node level)
- Port range limitation (30000–32767)
- Client must know node IP (no single stable endpoint)

---

## 5. LoadBalancer

### Definition
Provisions an **external cloud load balancer** (AWS ALB/NLB, GCP LB, Azure LB) that routes traffic into the cluster.

### Why Needed
Production-grade external traffic ingestion with a single stable IP.

### Use Cases
- Production APIs
- Web applications
- Any service needing internet access with HA

### YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"   # AWS NLB
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  loadBalancerSourceRanges:            # Restrict IPs allowed to connect
  - "203.0.113.0/24"
```

### What Happens Automatically
1. K8s creates a NodePort service underneath
2. Cloud controller manager provisions cloud LB
3. LB forwards to NodePort on healthy nodes
4. Service gets `EXTERNAL-IP` assigned

### Problem Without LoadBalancer (using NodePort instead)
- No automatic health checks at cloud level
- No single stable DNS/IP
- Node IP exposed publicly
- No TLS termination at cloud level
- No DDoS protection / rate limiting
- Client must handle failover manually

---

## 6. Headless Service

### Definition
A service with `clusterIP: None`. No virtual IP is assigned. DNS returns **individual Pod IPs** directly.

### Why Needed
When you need direct Pod-to-Pod connections (not load-balanced). Required for stateful apps where each pod has a unique identity.

### Use Cases
- StatefulSets (Kafka, Zookeeper, Cassandra, MySQL replication)
- Custom load balancing in the application layer
- Peer discovery (Elasticsearch nodes finding each other)

### YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
spec:
  clusterIP: None                       # KEY: makes this headless
  selector:
    app: kafka
  ports:
  - port: 9092
    targetPort: 9092
```

### DNS Behavior
```
# Normal ClusterIP:
kafka-svc.default.svc.cluster.local → 10.96.0.1 (virtual IP)

# Headless:
kafka-headless.default.svc.cluster.local → [10.0.0.5, 10.0.0.6, 10.0.0.7]

# Individual pod DNS (only with StatefulSet):
kafka-0.kafka-headless.default.svc.cluster.local → 10.0.0.5
kafka-1.kafka-headless.default.svc.cluster.local → 10.0.0.6
```

---

## 7. ExternalName

### Definition
Maps a service to an **external DNS name** using CNAME. No proxying, just DNS redirect.

### Why Needed
Allows pods to access external services using Kubernetes-native DNS names, making it easy to swap them later.

### Use Cases
- External managed database (RDS, Cloud SQL)
- Third-party APIs (Stripe, Twilio)
- Migrating services out of cluster gradually

### YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: default
spec:
  type: ExternalName
  externalName: mydb.us-east-1.rds.amazonaws.com   # CNAME target
```

### How It Works
```
Pod calls: external-db.default.svc.cluster.local
CoreDNS returns CNAME: mydb.us-east-1.rds.amazonaws.com
Pod resolves: actual RDS IP
```

> No kube-proxy involvement. Pure DNS-level redirect.

---

## 8. Services With No Selectors

### Definition
A Service with no `selector` field. K8s won't auto-create Endpoints — you define them manually.

### Why Needed
- Connect to external services by IP (no DNS)
- Connect to services in another namespace/cluster
- Legacy migrations (service in cluster, backend outside)
- Blue/green deployments pointing to different endpoint sets

### YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-legacy-db
spec:
  ports:
  - port: 5432
    targetPort: 5432
  # No selector — endpoints managed manually
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-legacy-db           # MUST match Service name exactly
subsets:
- addresses:
  - ip: 192.168.1.100                # External/legacy server IP
  - ip: 192.168.1.101
  ports:
  - port: 5432
```

---

## 9. Endpoints vs EndpointSlices

### Endpoints (Legacy)

**Definition:** An Endpoints object lists all Pod IPs + ports for a Service. Auto-created when you create a Service with a selector.

**Problem:**
- A single Endpoints object stores **all** IPs
- Default limit: **100 endpoints** (soft cap, practical limit)
- Explicit max: **1000 endpoints** (hard limit per object in etcd)
- Any single pod change → **entire Endpoints object updated** → massive etcd + API server churn
- At 5000 nodes × 1000 pods = etcd write storm

### EndpointSlices (Modern — GA since K8s 1.21)

**Definition:** Shards the endpoint data into **smaller slices**, each holding max **100 endpoints** by default.

**Why Built:**
- Large clusters with thousands of pods were hitting etcd limits
- One pod dying caused full Endpoints object rewrite
- EndpointSlices update only the **affected slice**

**Supports:** IPv4, IPv6, FQDN, internal IPs, external IPs

### Comparison Table

| Feature | Endpoints | EndpointSlices |
|---------|-----------|---------------|
| Max per object | 1000 | 100 (default) |
| IPv6 support | Limited | Native |
| Topology info | None | Zone/node aware |
| Update efficiency | Full rewrite | Partial slice update |
| Auto-created | Yes (by Service) | Yes (K8s 1.17+) |
| Recommended | Legacy only | Yes — default since 1.21 |

### EndpointSlice YAML

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-app-abc123                  # Auto-generated name
  namespace: default
  labels:
    kubernetes.io/service-name: my-app # Links slice to Service
addressType: IPv4                      # IPv4 | IPv6 | FQDN
endpoints:
- addresses:
  - "10.0.0.5"                         # Pod IP
  conditions:
    ready: true                        # Pod is ready to receive traffic
    serving: true                      # Pod is serving (even if terminating)
    terminating: false                 # Pod is NOT terminating
  nodeName: node-1                     # Which node this pod is on
  zone: us-east-1a                     # Availability zone (topology)
  targetRef:
    kind: Pod
    name: my-app-xyz
    namespace: default
- addresses:
  - "10.0.0.6"
  conditions:
    ready: false                       # Pod NOT ready — excluded from LB
    serving: false
    terminating: true                  # Pod is shutting down
  nodeName: node-2
ports:
- name: http
  port: 8080
  protocol: TCP
```

### Auto-Management Flow
```
Service Created
    ↓
EndpointSlice Controller watches pods matching selector
    ↓
Creates EndpointSlice(s) with ready pod IPs
    ↓
Pod dies → controller removes that IP from slice (only that slice updated)
    ↓
New pod ready → IP added to existing slice (or new slice if full)
```

### If Node Has 5 Pods, 2 Are Dead
```
- Controller detects 2 pods: conditions.ready = false
- Those 2 IPs removed from EndpointSlice
- Traffic only routes to 3 healthy pods
- No new pods created (that's Deployment's job, not EndpointSlice)
- EndpointSlice ONLY reflects current ready state
```

> **Key Insight:** EndpointSlice does NOT create pods. It REFLECTS pod readiness. Deployment controller creates new pods when replicas are below desired count.

### CPU/Traffic Scaling Connection
```
High CPU on pods
    ↓
HPA scales up Deployment → more pods created
    ↓
New pods become Ready
    ↓
EndpointSlice controller adds new pod IPs to slice
    ↓
kube-proxy updates iptables/IPVS
    ↓
Traffic distributed to more pods
```

---

## 10. Internal Pod vs External Pod Communication

### Internal Pod Communication (Same Cluster)

```
Pod A (10.0.0.5) → Service DNS → ClusterIP → iptables/IPVS → Pod B (10.0.0.8)
```

- Uses CoreDNS for service discovery
- Flat network — all pods can reach all pods by IP
- CNI plugin (Calico, Flannel, Cilium) manages pod network

### External to Pod Communication

```
Internet → Cloud LB (External IP) → NodePort → Service → Pod
Internet → Ingress Controller Pod → Service ClusterIP → Pod
```

### Internal Pod → External Service

```
Pod → Service (ExternalName/No-selector Endpoints) → External IP/DNS
Pod → Direct external IP (via NAT on node)
```

### Communication Matrix

| Source | Destination | Method |
|--------|------------|--------|
| Pod → Pod (same node) | Direct via veth/bridge | CNI |
| Pod → Pod (diff node) | CNI overlay (VXLAN/BGP) | CNI |
| Pod → Service | ClusterIP + iptables | kube-proxy |
| External → Service | LoadBalancer/NodePort | cloud LB |
| Pod → External | SNAT on node | iptables masquerade |

---

## 11. IPs — Internal, External, Cloud Rules

### Pod Internal IP
- Assigned by CNI from pod CIDR (e.g., `10.244.0.0/16`)
- Only routable **within cluster**
- Changes every time pod restarts

### Node Internal IP
- Node's actual network IP (VPC internal)
- Stable, used by NodePort

### Service ClusterIP
- Virtual IP from service CIDR (e.g., `10.96.0.0/12`)
- Never actually bound to any interface
- Implemented via iptables/IPVS rules

### External IP (LoadBalancer)
- Provisioned by cloud controller
- Routes to NodePort on healthy nodes
- DNS A-record typically points here

### Cloud Rules Used (AWS Example)

| Concept | AWS Equivalent |
|---------|---------------|
| LoadBalancer Service | NLB or CLB |
| Ingress (nginx) | Application Load Balancer (ALB) |
| ExternalName | Route53 CNAME |
| NodePort | Security Group rule on port |
| Pod IP | ENI secondary IP (with VPC CNI) |

---

## 12. WebSocket & LoadBalancer Considerations

### WebSocket Problem
WebSocket requires **persistent long-lived TCP connections**. Standard HTTP load balancers that terminate connections frequently will break WebSocket.

### Problems If You Don't Use a Proper LoadBalancer

| Problem | Impact |
|---------|--------|
| Connection timeout | WebSocket disconnected mid-session |
| Round-robin LB | Client reconnects to different pod → lost session state |
| HTTP/1.1 keep-alive not honored | Frequent reconnects |
| No sticky sessions | Message ordering broken |

### Solutions

```yaml
# Service with session affinity (sticky sessions)
apiVersion: v1
kind: Service
metadata:
  name: websocket-svc
spec:
  type: LoadBalancer
  selector:
    app: websocket-app
  sessionAffinity: ClientIP            # Same client → same pod
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800            # 3 hour session
  ports:
  - port: 443
    targetPort: 8080
```

For Ingress (nginx):
```yaml
nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
nginx.ingress.kubernetes.io/upstream-hash-by: "$remote_addr"   # sticky
```

---

## 13. Load Balancer Internal Topology Algorithms

### kube-proxy Modes

#### iptables Mode (default)
- Random selection among healthy endpoints
- Uses iptables probability rules
- Stateless — no connection tracking for LB decisions
- O(n) rule scan per packet

#### IPVS Mode (recommended for large clusters)
- Kernel-level load balancing
- Multiple algorithms supported
- O(1) lookup

### IPVS Algorithms

| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| `rr` (Round Robin) | Equal distribution | Stateless uniform pods |
| `lc` (Least Connection) | Fewest active connections | Long-lived connections |
| `dh` (Destination Hash) | Hash of destination IP | Cache locality |
| `sh` (Source Hash) | Hash of source IP | Session stickiness |
| `sed` (Shortest Expected Delay) | Weighted by connections | Mixed capacity pods |
| `nq` (Never Queue) | No idle server waits | Latency-sensitive |

### Topology-Aware Routing (K8s 1.23+)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    service.kubernetes.io/topology-mode: "Auto"   # Prefer same-zone pods
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

- Routes traffic to pods **in the same availability zone first**
- Reduces cross-zone data transfer costs
- Falls back to other zones if local zone has no ready pods
- EndpointSlice `zone` field used for this

---

## 14. Ingress — Full Deep Dive

### Definition
**Ingress** is a K8s API object that defines **HTTP/HTTPS routing rules** (L7). It is NOT itself a load balancer — it's just a config object.

### Architecture
```
Internet
    ↓
WAF (optional) — L7 filtering
    ↓
Cloud LoadBalancer (L4/L7) — External IP
    ↓
Ingress Controller Pod (nginx/traefik/etc) — L7 routing
    ↓
Service (ClusterIP) — L4
    ↓
Pod
```

### Why Ingress Over LoadBalancer-Per-Service?
- One LoadBalancer IP for all services (cost saving — cloud LBs are expensive)
- Path-based and domain-based routing at L7
- TLS termination at one place
- Centralized auth, rate limiting, WAF

### Full Ingress YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /         # Rewrite URL path
    nginx.ingress.kubernetes.io/ssl-redirect: "true"      # Force HTTPS
    nginx.ingress.kubernetes.io/use-regex: "true"         # Allow regex paths
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"    # Max request size
    nginx.ingress.kubernetes.io/rate-limit: "100"         # Requests/minute
spec:
  ingressClassName: nginx                                  # Which controller handles this
  tls:
  - hosts:
    - app.example.com
    - api.example.com
    secretName: tls-secret                                # K8s Secret with cert+key
  rules:
  # Domain-based routing
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
  # Path-based routing on same domain
  - host: api.example.com
    http:
      paths:
      - path: /v1/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      - path: /v1/orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /                                            # Default fallback
        pathType: Prefix
        backend:
          service:
            name: default-backend
            port:
              number: 80
```

---

## 15. Ingress Controller vs Ingress Object

### Ingress Object
- Just a **configuration** — declarative routing rules
- Stored in etcd
- Does nothing by itself
- Matches: IP, domain, path

### Ingress Controller
- An actual **running Pod** that watches Ingress objects
- Reads rules and configures itself (nginx.conf, etc.)
- **Routes actual traffic**
- Examples: nginx, traefik, HAProxy, Istio Gateway, AWS ALB Controller

### Relationship
```
Ingress Object (rules) ──watches──► Ingress Controller (nginx pod)
                                           │
                                    configures nginx.conf
                                           │
                               actual HTTP routing to Services
```

### nginx Ingress Controller Deploy

```yaml
# IngressClass — tells controller which Ingress objects to handle
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"   # Default for all Ingress
spec:
  controller: k8s.io/ingress-nginx
```

### 404 Fast Debug
```bash
# Check Ingress object
kubectl get ingress -n <namespace>
kubectl describe ingress <name>

# Check Ingress Controller logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=100

# Check if IngressClass matches
kubectl get ingressclass

# Check if backend Service exists
kubectl get svc <backend-service-name>

# Check if Endpoints are populated
kubectl get endpoints <service-name>

# Test from inside cluster
kubectl run debug --image=curlimages/curl -it --rm -- curl http://<service>:<port>
```

---

## 16. Path-Based, Domain-Based, TLS Ingress

### Path-Based Routing
One domain, multiple paths → different services
```
api.example.com/users   → user-service:80
api.example.com/orders  → order-service:80
api.example.com/        → default-backend:80
```

### Domain-Based Routing
Multiple domains → different services (virtual hosting)
```
app.example.com    → frontend-svc:80
api.example.com    → backend-svc:80
admin.example.com  → admin-svc:80
```

### Path Types

| Type | Behavior |
|------|---------|
| `Exact` | Exact URL match only |
| `Prefix` | Matches path prefix (most common) |
| `ImplementationSpecific` | Controller decides |

### TLS Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
```

```bash
# Generate via cert-manager (recommended)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Or from existing cert files
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

---

## 17. DNS — A Record, CNAME, IngressClassName

### A Record
- Maps domain name → IPv4 address
- Points to LoadBalancer's External IP
```
app.example.com → A → 34.100.200.50 (LB external IP)
```

### CNAME
- Maps domain name → another domain name
- Used for ExternalName services, CDN, cloud services
```
app.example.com → CNAME → my-lb.us-east-1.elb.amazonaws.com
```

### When to Use Which
| Scenario | Use |
|---------|-----|
| Static IP from cloud LB | A Record |
| AWS ELB (has DNS name, not IP) | CNAME |
| ExternalName Service | CNAME |
| Jio/ISP infra with stable IP | A Record |

### IngressClassName
```yaml
spec:
  ingressClassName: nginx   # Must match IngressClass object name
```
- Tells K8s **which Ingress Controller** should process this Ingress
- If wrong/missing → 404 (controller ignores the Ingress)
- Multiple controllers can run simultaneously (nginx + traefik)

---

## 18. WAF (Web Application Firewall) with Ingress

### Definition
WAF inspects HTTP/HTTPS traffic at L7 for attacks: SQL injection, XSS, CSRF, DDoS, bot traffic.

### Traffic Flow with WAF
```
Internet
    ↓
WAF (e.g., AWS WAF, Cloudflare, ModSecurity)   ← L7 filtering
    ↓
Cloud Load Balancer                              ← L4/L7 external entry
    ↓
Ingress Controller (nginx pod)                   ← L7 routing rules
    ↓
Service (ClusterIP)                              ← L4 stable IP
    ↓
Pod                                              ← App container
```

### ModSecurity with nginx Ingress

```yaml
# Enable WAF via annotations
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-core-rules: "true"
    nginx.ingress.kubernetes.io/modsecurity-snippet: |
      SecRuleEngine On
      SecRequestBodyAccess On
```

### External WAF (AWS WAF Example)

```yaml
# AWS ALB Ingress with WAF
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:...
    alb.ingress.kubernetes.io/scheme: internet-facing
```

---

## 19. Full Traffic Flow Diagram

```
                          ┌─────────────────────────────────────────────────────┐
                          │                   KUBERNETES CLUSTER                │
                          │                                                     │
   Internet               │    ┌──────────────────────────────────────────┐    │
      │                   │    │              NODE 1 (192.168.1.10)        │    │
      ▼                   │    │                                          │    │
 ┌─────────┐              │    │  ┌────────────────┐  ┌────────────────┐  │    │
 │   WAF   │              │    │  │ nginx-ingress  │  │   app-pod-1    │  │    │
 └────┬────┘              │    │  │   (pod)        │  │  (10.0.0.5)   │  │    │
      │                   │    │  └───────┬────────┘  └────────────────┘  │    │
      ▼                   │    │          │                                │    │
 ┌─────────┐              │    └──────────┼────────────────────────────────┘    │
 │Cloud LB │──────────────►               │                                     │
 │Ext IP   │              │    ┌──────────▼────────────────────────────────┐    │
 └─────────┘              │    │              NODE 2 (192.168.1.11)        │    │
                          │    │                                          │    │
                          │    │  ┌────────────────┐  ┌────────────────┐  │    │
                          │    │  │  ClusterIP Svc │  │   app-pod-2    │  │    │
                          │    │  │  10.96.0.1:80  │  │  (10.0.0.6)   │  │    │
                          │    │  └───────┬────────┘  └────────────────┘  │    │
                          │    │          │                                │    │
                          │    │    EndpointSlice: [10.0.0.5, 10.0.0.6]  │    │
                          │    └──────────────────────────────────────────┘    │
                          └─────────────────────────────────────────────────────┘

Flow:
User → WAF → Cloud LB → NodePort:30080 → Ingress Controller Pod
 → ClusterIP Service (10.96.0.1:80)
 → kube-proxy reads EndpointSlice
 → routes to Pod (10.0.0.5:8080 or 10.0.0.6:8080)
```

---

## 20. Interview Questions & Answers

### Q1: What is the difference between ClusterIP, NodePort, and LoadBalancer?
**A:** ClusterIP is only accessible inside the cluster — it's a virtual IP for internal service communication. NodePort opens a static port (30000-32767) on every node, making the service accessible externally via `<NodeIP>:<NodePort>`. LoadBalancer provisions a cloud-provided external load balancer with a public IP, which is the production-grade way to expose services to the internet. LoadBalancer builds on top of NodePort, which builds on top of ClusterIP.

---

### Q2: What is a Headless Service and when would you use it?
**A:** A headless service has `clusterIP: None`. Instead of a virtual IP, DNS resolves directly to individual Pod IPs. You'd use it for StatefulSets (Kafka, Cassandra, ZooKeeper) where each pod has a stable identity and you need direct pod-to-pod communication, or when your application handles its own load balancing/peer discovery.

---

### Q3: What is the difference between Endpoints and EndpointSlices?
**A:** Endpoints is the legacy object — one object stores ALL pod IPs for a service. This causes problems at scale: 1000+ pods means one massive object, and any pod change rewrites the entire thing (etcd write storm). EndpointSlices shard this into 100-endpoint chunks. Only the affected slice is updated when a pod changes. EndpointSlices also support IPv6, topology zones, and termination state. Default since K8s 1.21.

---

### Q4: Can you create a Service with no selectors? Why would you?
**A:** Yes. Without a selector, K8s won't auto-create Endpoints — you create them manually. Use cases: connecting to external databases by IP, connecting to services outside the cluster, legacy system integration, or pointing to a different namespace. You'd pair the no-selector Service with a manual Endpoints or EndpointSlice object with the same name.

---

### Q5: What is the default max number of endpoints?
**A:** By default, K8s truncates Endpoints objects at **1000** entries (configurable via `--max-endpoints-per-slice` on the controller). The "soft" practical limit before performance degrades is around 100 (which is why EndpointSlice uses 100 as its chunk size).

---

### Q6: What happens to EndpointSlice when 2 out of 5 pods die?
**A:** The EndpointSlice controller detects that those 2 pods are no longer Ready (readinessProbe fails or pod is deleted). It updates their `conditions.ready` to `false` and removes them from the serving set. Traffic is automatically routed to only the 3 healthy pods. The EndpointSlice does NOT create new pods — that's the Deployment/ReplicaSet controller's job.

---

### Q7: Explain port, targetPort, and nodePort in a NodePort Service.
**A:** `nodePort` (30000-32767) is the port opened on every Node's IP — external clients connect here. `port` is the ClusterIP-level port — internal cluster traffic uses this. `targetPort` is the actual container port your application listens on. If `targetPort` is omitted, it defaults to the same value as `port`. So: `ExternalClient:nodePort → Service:port → Container:targetPort`.

---

### Q8: What is an IngressClass and why is it important?
**A:** IngressClass is a resource that links an Ingress object to a specific Ingress Controller. If you have both nginx and traefik running, `ingressClassName: nginx` tells K8s that the nginx controller should process this Ingress. Without it (or with the wrong value), the controller ignores your Ingress and you get 404s.

---

### Q9: What is the difference between an Ingress Object and an Ingress Controller?
**A:** Ingress Object is just a declarative YAML config — it defines routing rules (which path/domain goes to which service). It's stored in etcd and does nothing by itself. Ingress Controller is an actual running pod (nginx, traefik, etc.) that watches Ingress objects, reads the rules, and configures itself to actually route HTTP traffic. Object = rules; Controller = enforcement.

---

### Q10: Why use Ingress instead of a LoadBalancer per service?
**A:** Cloud LoadBalancers cost money — one per service adds up fast. Ingress allows one LB to serve all HTTP/HTTPS services via path-based and domain-based routing. You also get centralized TLS termination, WAF, auth, and rate limiting. The tradeoff is that Ingress only handles L7 (HTTP/HTTPS); non-HTTP services still need their own LB or NodePort.

---

### Q11: How does topology-aware routing work in EndpointSlices?
**A:** Each endpoint in an EndpointSlice has a `zone` field (e.g., `us-east-1a`). With `service.kubernetes.io/topology-mode: Auto`, kube-proxy prefers routing traffic to pods in the same availability zone as the requesting node. This reduces cross-zone latency and cloud data transfer costs. It falls back to any zone if the local zone has no ready pods.

---

### Q12: What are the problems with WebSocket on standard LoadBalancers?
**A:** WebSocket needs persistent, long-lived TCP connections. Problems arise when: (1) LB has short idle timeouts (connection dropped mid-session), (2) LB doesn't honor HTTP Upgrade header, (3) Round-robin breaks session continuity — client reconnects to different pod and loses state. Solutions: Use `sessionAffinity: ClientIP` in Service, set long proxy timeouts in nginx Ingress annotations, or use sticky sessions.

---

### Q13: What is ExternalName service and how does it differ from regular services?
**A:** ExternalName maps a K8s service name to an external DNS name via CNAME — no kube-proxy, no iptables, pure DNS redirect. A regular service (ClusterIP etc.) proxies traffic through kube-proxy/IPVS. ExternalName just returns a CNAME so the client resolves the external DNS itself. Use for external managed DBs, third-party APIs, or gradual migrations.

---

### Q14: Describe the full request path: Internet → Pod.
**A:** Internet → WAF (optional L7 filtering) → Cloud LoadBalancer (external IP, L4/L7) → Node NodePort → iptables/IPVS on node → Service ClusterIP (virtual IP) → kube-proxy looks up EndpointSlice → selects healthy pod IP → packet routed via CNI to pod container → application processes request.

---

### Q15: What IPVS algorithms does K8s support and when would you choose each?
**A:** Round Robin (rr) for stateless uniform pods; Least Connection (lc) for varying connection durations; Source Hash (sh) for session stickiness; Destination Hash (dh) for cache locality; Shortest Expected Delay (sed) for mixed-capacity nodes. Enable IPVS with `--proxy-mode=ipvs` on kube-proxy — it offers O(1) lookup vs iptables' O(n).

---

### Q16: How does automatic EndpointSlice update work when CPU is high and HPA scales up?
**A:**
1. HPA detects CPU > threshold via metrics-server
2. HPA increases Deployment replicas
3. Deployment creates new Pods
4. Pods pass readinessProbe → status = Ready
5. EndpointSlice controller detects new ready pods
6. Adds new pod IPs to EndpointSlice (new slice created if existing is full at 100)
7. kube-proxy on all nodes gets update via watch API
8. iptables/IPVS updated to include new pod IPs
9. Traffic now distributed across more pods

---

### Quick Reference Cheat Sheet

```
kubectl get svc                          # List all services
kubectl describe svc <name>             # Service details + endpoints
kubectl get endpoints                   # Legacy endpoints
kubectl get endpointslices              # Modern endpoint slices
kubectl get ingress                     # List ingress objects
kubectl describe ingress <name>         # Ingress rules + backend status
kubectl get ingressclass                # Available controllers
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller  # Ingress logs
kubectl run test --image=curlimages/curl -it --rm -- sh        # Debug pod
```

---

*Generated for Kubernetes Networking Deep Dive Study — covers K8s 1.21+ features*
*Focus: CKA/CKAD/Production Interview Preparation*
