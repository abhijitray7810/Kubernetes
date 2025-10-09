# Nginx Deployment on Kubernetes

This repository contains a Kubernetes **Deployment** manifest for running **Nginx** in the `nginx` namespace.

---

## 📄 Deployment Details

- **API Version**: `apps/v1`
- **Kind**: `Deployment`
- **Name**: `nginx-deployment`
- **Namespace**: `nginx`
- **Replicas**: `2`
- **Labels**: `app: nginx`
- **Container**:
  - **Image**: `nginx:latest`
  - **Port**: `80`

---

## 🚀 How to Deploy

1. Ensure you have a running Kubernetes cluster and `kubectl` configured.

2. Create the namespace (if it doesn’t exist):
   ```bash
   kubectl create namespace nginx
   ```
Apply the deployment:

```bash

kubectl apply -f nginx-deployment.yaml
```
# 📊 Verify the Deployment
Check the deployment status:

```bash

kubectl get deployments -n nginx
```
Check the pods:

bash
Copy code
kubectl get pods -n nginx
Describe the deployment:

```bash

kubectl describe deployment nginx-deployment -n nginx
```
# 🌐 Accessing Nginx
By default, this deployment only creates Pods.
To expose Nginx, create a Service:

yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
Apply it:

```bash

kubectl apply -f nginx-service.yaml
```
Then you can access the service inside the cluster:

```bash

kubectl get svc -n nginx
```
# 🧹 Cleanup
To remove all resources:

```bash

kubectl delete -f nginx-deployment.yaml

```
```bash
kubectl delete -f nginx-service.yaml
```
```bash
kubectl delete namespace nginx
```
```
kubectl scale deployment/nginx-deployment -n nginx --replicas=1,5,41,2,3
```
abhi@OMEN16:~/Kubernetes/kube-02/nginx$ kubectl scale deployment/nignx-deployment -n nginx --replicas=5
deployment.apps/nignx-deployment scaled
```
kubectl get pods -n nginx
```
abhi@OMEN16:~/Kubernetes/kube-02/nginx$ kubectl get pods -n nginx
NAME                                READY   STATUS    RESTARTS   AGE
nignx-deployment-5f7bc67bd9-86zbt   1/1     Running   0          36s
nignx-deployment-5f7bc67bd9-hrnvn   1/1     Running   0          10m
nignx-deployment-5f7bc67bd9-ljzvc   1/1     Running   0          10m
nignx-deployment-5f7bc67bd9-lt7cl   1/1     Running   0          36s
nignx-deployment-5f7bc67bd9-pzvtv   1/1     Running   0          36s
