# 🐳 Kubernetes DaemonSet: Nginx Deployment

This repository contains a **Kubernetes DaemonSet manifest** that ensures an **Nginx container** runs on **all nodes** in the cluster.

---

## 📌 What is a DaemonSet?

- A **DaemonSet** ensures that a **Pod** is scheduled on **every node** in the cluster.  
- Common use cases: logging agents, monitoring agents, and services like Nginx that should be available everywhere.  
- If a new node is added, the DaemonSet automatically schedules a pod on it.  

---

## 📜 Manifest: `nginx-daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
  namespace: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80

```
# 🛠 Explanation of the Manifest
kind: DaemonSet → Ensures one Nginx pod per node.

metadata.name → Name of the DaemonSet (nginx-daemonset).

namespace: nginx → Deploys in the nginx namespace (must be created first).

selector & labels → Ensure pods are managed by this DaemonSet.

containers → Runs the latest Nginx image and exposes port 80 inside the pod.

# ▶️ Steps to Deploy
Create the namespace (if it doesn’t exist):

```bash
kubectl create namespace nginx

```
Apply the DaemonSet manifest:

```bash

kubectl apply -f nginx-daemonset.yaml
```
Verify DaemonSet:

```bash

kubectl get daemonset -n nginx
```
Example output:

```pgsql

NAME              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
nginx-daemonset   3         3         3       3            3           <none>          10s
```
Check Pods:
```bash

kubectl get pods -n nginx -o wide
```
# ✔️ You should see one Nginx pod per node in the cluster.

# 📋 Testing Nginx
To test Nginx, you can expose one of the pods or use kubectl port-forward:

```bash

kubectl port-forward -n nginx pod/<nginx-pod-name> 8080:80
```
Now visit:



http://localhost:8080
# ✔️ You should see the Nginx welcome page 🎉

# 🧹 Cleanup
To delete the DaemonSet:

```bash

kubectl delete -f nginx-daemonset.yaml
```

# ✅ Summary
Created an Nginx DaemonSet that runs on every node

Verified deployment using kubectl get daemonset

Tested by accessing the Nginx pods locally

# 🎯 With this setup, Nginx will always run across your Kubernetes cluster.

👉 Do you want me to also add a **Service object (NodePort / ClusterIP / LoadBalancer)** in the READ
