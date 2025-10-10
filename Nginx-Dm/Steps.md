## Applying
```
vi Deployment.yaml
```
## Applying
```
kubectl apply -f Deployment.yml
```
## Scale
```
kubectl scale deployment/nginx-deployment -n nginx --replicas=5
```
## Pods Creation
```
kubectl get pods -n nginx
```
## More Detilas
```
kubectl get pods -n nginx -o wide
```
## if you update any image version
```
Kubectl set image deployment/nginx deployment -n nginx nginx=nginx:1.27.3
```
this commands are using if you any version of image then this is using saffication only  
