## First Namespace start
```
kubectl create namespace nginx
```
## Apply 
```
kubectl apply -f namespace.yml
```
## Cron Job Apply
```
kubectl apply -f cron-job.yml
```
output:
```
abhi@OMEN16:~/Kubernetes/kube-02/nginx$ kubectl apply -f cron-job.yml
cronjob.batch/every-minute-backup created
```
## verify
```
kubectl -n nignx get cron-job
```
output:
```
abhi@OMEN16:~/Kubernetes/kube-02/nginx$ kubectl -n nginx get cj
NAME                  SCHEDULE    TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
every-minute-backup   * * * * *   <none>     False     0        26s             44s
```
## Jobs-Watch
```
kubectl -n nginx get jobs --watch
```
## Delete cronjob
```
kubectl delete -f cron-job.yml
```
## Pods
```
abhi@OMEN16:~/Kubernetes/kube-02/nginx$ kubectl get pods -n nginx
NAME                                 READY   STATUS      RESTARTS   AGE
every-minute-backup-29339345-ftrzh   0/1     Completed   0          37s
```
