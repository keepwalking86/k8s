## Persistent storage use GlusterFS

Yêu cầu:

Hệ thống đã cài đặt GlusterFS với 02 nodes với GlusterFS replication

- Gluster01: 192.168.10.247

- Gluster02: 192.168.10.248

- Gluster-VIP: 192.168.10.249

- Volume name: media_replica

Trên các node trong k8s cluster, cài đặt thư viện glusterfs-fuse

**Step1: Create an Gluster Endpoint for the gluster service**

Gluster endpoint định nghĩa các node trong Gluster-trusted storage pool. Trong trường hợp hệ thống GlusterFS sử dụng kiểu Replicated, khi đó chỉ cần khai báo địa chỉ IP VIP của glusterfs storage.

```
#cat gluster-endpoints.yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: glusterfs-cluster 
subsets:
  - addresses:
      - ip: 192.168.10.249
    ports:
      - port: 1
```

Trong đó:

- glusterfs-cluster là name của endpoint, và cùng name với service

- port khai báo cùng port với service

- 192.168.10.249 là địa chỉ IP VIP của cụm GlusterFS storage

- port thì khai báo port gì cũng được, nhưng thường port để là 1

**Create endpoint**

`kubectl create -f gluster-endpoints.yaml`

Check endpoints

[root@master1 gluster_pod]# kubectl get endpoints

|NAME                |ENDPOINTS                                                     |AGE|
|--------------------|--------------------------------------------------------------|---|
|default-subdomain   |<none>                                                        |19d|
|glusterfs-cluster   |192.168.10.249:1                                              |6h|
|kubernetes          |192.168.10.222:6443,192.168.10.223:6443,192.168.10.224:6443   |21d|

**Step2: Create service**

Một gluster service được tạo ra để giữ cho gluster endpoint cố định

Tạo tệp tin gluster-service.yaml

```
#cat gluster-service.yaml
apiVersion: "v1"
kind: "Service"
metadata:
  name: "glusterfs-cluster"
spec:
  ports:
  - port: 1
```

Trong đó:

- glusterfs-cluster là tên của service, khi đó endpoint name phải cùng tên với service name.

- port cùng với port khai báo với endpoint

**Create service**

`kubectl create -f gluster-service.yaml`

**Step3: Creating the Persistent Volume**

Định nghĩa một Persistent Volume (pv) với tên pv gồm thông tin gluster volume name, dung lượng cấp phát, chế độ truy cập volume

```
#cat gluster-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gluster-volume-media
spec:
  capacity:
    storage: 2Gi 
  accessModes: 
    - ReadWriteMany
  glusterfs: 
    endpoints: glusterfs-cluster 
    path: media_replica
    readOnly: false
  persistentVolumeReclaimPolicy: Delete
```

Trong đó:

- gluster-volume-media là tên của persistent volume

- storage: 2Gi là lượng lưu trữ được cấp phát cho volume

- accessModes: Thiết lập chế độ truy cập storage ở phạm vi volume này.

GlusterFS hỗ trợ 3 chế độ truy cập cho Persistent Volume: ReadWriteOnce(RWO), ReadOnlyMany(ROX) và ReadWriteMany (RWX)

- Khai báo loại Persistent Volume. Ở đây, chúng ta sử dụng glusterfs

- path: khai báo **volume name** định nghĩa trong GlusterFS. Ở đây, chúng ta sử dụng volume name là **media_replica** (hoặc
/media_replica)

- persistentVolumeReclaimPolicy: chính sách lấy lại Persistent Volume. Giá trị có thể tùy chọn Retain hoặc Delete và Recycle (deprecated)

**Create persistent volume**

`kubectl create -f gluster-pv.yaml`

check persistent volume

`kubectl get pv`

|NAME                   |CAPACITY   |ACCESS MODES   |RECLAIM POLICY   |STATUS      |CLAIM   |STORAGECLASS   |REASON   |AGE|
|-----------------------|-----------|---------------|-----------------|------------|--------|---------------|---------|---|
|gluster-volume-media   |2Gi        |RWX            |Delete           |Available   |        |               |         |8s |


**Step4: Create Persistent Volume Claim**

Định nghĩa một Persistent Volume Claim (PVC) với dung lượng cấp phát và chế độ truy cập mà pod có thể sử dụng. Một PVC được liên kết với chỉ một
PV.

Tạo tệp tin gluster-pvc.yaml
```
#cat gluster_pod/gluster-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gluster-claim
spec:
  accessModes:
  - ReadWriteMany
  resources:
     requests:
       storage: 2Gi
  volumeName: gluster-volume-media
```

Trong đó:

- gluster-claim là tên của claim được định nghĩa

- volumeName: gluster-default-volume. Nếu muốn gán chính xác PersistentVolume cho PersistentVolumeClaim khi đó sử dụng trường *volumeName*

**Create Persistent Volume Claim**

`kubectl create -f gluster-pvc.yaml`

Kiểm tra pvc vừa tạo

`kubectl get pvc`

|NAME            |STATUS   |VOLUME                 |CAPACITY   |ACCESS MODES   |STORAGECLASS   AGE|
|----------------|---------|-----------------------|-----------|---------------|------------------|
|gluster-claim   |Bound    |gluster-volume-media   |2Gi        |RWX            |               37s|


**Step5: Using pvc in pod**

Định nghĩa một pod sử dụng nginx image và để sử dụng PVC vừa tạo

```
#cat gluster_pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gluster-pod1
  labels:
    name: gluster-pod1
spec:
  securityContext:
    supplementalGroups: [1000]
    fsGroup: 1000
  containers:
  - name: gluster-pod1
    image: nginx
    ports:
    - name: web
      containerPort: 80
    securityContext:
      privileged: true
    volumeMounts:
    - name: gluster-web1
      subPath: pod1
      mountPath: /usr/share/nginx/html
      readOnly: false
  volumes:
  - name: gluster-web1
    persistentVolumeClaim:
      claimName: gluster-claim
```

Trong đó:

- gluster-pod1 là name của pod

- sử dụng image nginx cho pod

- gluster-web1 là tên volume sử dụng cho pod này. Tên volume này cùng tên trong cả phần volume và containers.

- subPath: web1 . Với mỗi pod, chúng ta sẽ mount vào sub directory của glusterfs volume. Ở đây, chúng ta mount /usr/share/nginx/html của pod vào subdir là web1 của
**media_replica** volume.

- gluster-claim là tên pvc đã khai báo ở phần tạo pvc

- Ngoài ra còn thiết lập phần security truy cập volume, với các tùy chọn trong tham số **securityContext**

**Thực hiện tạo pod**

`kubectl create -f gluster-pod.yaml`
