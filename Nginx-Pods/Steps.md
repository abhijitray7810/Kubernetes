# Describe Pods
```
kubectl describe pod/nginx-pod -n nginx
```
abhi@OMEN16:~/Kubernetes/kube-02/nginx$ kubectl describe pod/nginx-pod -n nginx
Name:             nginx-pod
Namespace:        nginx
Priority:         0
Service Account:  default
Node:             abhi-kube-worker3/172.18.0.4
Start Time:       Thu, 09 Oct 2025 19:00:52 +0000
Labels:           <none>
Annotations:      <none>
Status:           Running
IP:               10.244.3.3
IPs:
  IP:  10.244.3.3
Containers:
  nginx:
    Container ID:   containerd://ffa6773bcb1a01b74e870d07c771c5500c668ec8824c7ccbd1afa55280faca9c
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:3b7732505933ca591ce4a6d860cb713ad96a3176b82f7979a8dfa9973486a0d6
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 09 Oct 2025 19:00:56 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-tp4k2 (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  kube-api-access-tp4k2:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  14m   default-scheduler  Successfully assigned nginx/nginx-pod to abhi-kube-worker3
  Normal  Pulling    14m   kubelet            Pulling image "nginx:latest"
  Normal  Pulled     14m   kubelet            Successfully pulled image "nginx:latest" in 2.564s (2.564s including waiting). Image size: 62706233 bytes.
  Normal  Created    14m   kubelet            Created container: nginx
  Normal  Started    14m   kubelet            Started container nginx
