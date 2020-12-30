# Setup Rancher

[https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/](https://rancher.com/docs/rancher/v2.x/en/installation/install-rancher-on-k8s/)

Rancher là công cụ nguồn mở, sử dụng để quản lý Kubernetes cluster bằng giao diện web.

Chúng ta có thể cài đặt rancher trên một server độc lập với kubernetes cluster hoặc deploy như một service trong kubernetes cluster

## Installing rancher trên kubernetes cluster (k8s)

Yêu cầu:

- Đã cài đặt cert-manager

- Trỏ domain rancher cần dùng . Với trường hợp truy cập rancher từ internet, chúng ta trỏ domain vào địa chỉ public của kubernetes. Trong trường hợp chỉ cần truy cập từ local, chúng ta fix host đến external private IP của kubernetes.

**Sử dụng helm cài đặt rancher như sau**

```
[root@master1 ~]# helm install rancher rancher-latest/rancher   --namespace cattle-system   --set hostname=rancher.example.com --set ingress.tls.source=letsEncrypt   --set  letsEncrypt.email=keepwalking@example.com
```

```
W1208 00:30:01.728360    1650 warnings.go:67] extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
W1208 00:30:02.184427    1650 warnings.go:67] extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
NAME: rancher
LAST DEPLOYED: Tue Dec  8 00:30:01 2020
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.

NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued and Ingress comes up.

Check out our docs at https://rancher.com/docs/rancher/v2.x/en/

Browse to https://rancher.example.com

Happy Containering!
```

Hoặc sử dụng rancher-generated certificate như sau:

```
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com
```

Sử dụng trình duyệt để login rancher https://rancher.example.com

<p align="center">
<image src="../images/the-first-login-rancher.png" />
</p>

Thiết lập mật khẩu login lần đầu tiên và nhấn *Continue* để truy cập vào giao diện quản trị rancher

Local dashbard

<p align="center">
<image src="../images/rancher-dashboard1.png" />
</p>

Cluster Dashboard

<p align="center">
<image src="../images/rancher-dashboard2.png" />
</p>

