**Step1: Build image**

`docker build -t hub.example.com/laravel-kubernetes .`

**Step2: Push image**

`docker push hub.example.com/laravel-kubernetes`

**Step3: Deploy laravel to k8s**

`kubectl -f k8s/`


