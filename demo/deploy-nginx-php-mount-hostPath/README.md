# Deploy nginx php-fpm and mount code to hostPath

- Secret
- ConfigMap
- Deployment (nginx-php) with local mount (create /code directory all worker nodes -or- mount /code volume from storage as glusterfs)
- Service NodePort

**Step1: Build and push image to private registry**

```
docker build . -t hub.example.com/keepwalking86-nginx-php:latest
docker push hub.example.com/keepwalking86-nginx-php:latest
```

**Step2: Create /code on all worker nodes**

**Step3: Deploy on manifests to k8s**

`kubectl apply -f k8s/`

Warning: security risk from [https://kubernetes.io/docs/concepts/storage/volumes/#hostpath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)