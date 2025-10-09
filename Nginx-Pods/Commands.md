# Pods Creation
## Create pods.yml file
```
vi Pods.yml
```
## Apply pods
```
kubectl apply -f pods.yml
```
abhi@OMEN16:~/Kubernetes/kube-02/nginx$ vi pods.yml
abhi@OMEN16:~/Kubernetes/kube-02/nginx$ kubectl apply -f pods.yml
pod/nginx-pod created
## Get Pods
```
kubectl get pods -n nginx
```
abhi@OMEN16:~/Kubernetes/kube-02/nginx$ kubectl get pods -n nginx
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          19s
## kubectl exec
```
kubectl exec -it nginx-pod -n nginx -- bash
```
root@nginx-pod:/# ls
bin                   etc    mnt           run   usr
boot                  home   opt           sbin  var
dev                   lib    proc          srv
docker-entrypoint.d   lib64  product_uuid  sys
docker-entrypoint.sh  media  root          tmp


## curl
```
curl 127.0.0.1
```
root@nginx-pod:/# curl 127.0.0.1
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

