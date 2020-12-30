# Setup K8S on CentOS 7

**Phân hoạch địa chỉ IP cho các node như sau:**

|NAME              |IP               |ROLE  |
|------------------|-----------------|------|
|Master1           |192.168.10.222   |Master|
|Master2           |192.168.10.223   |Master|
|Master3           |192.168.10.224   |Master|
|Worker1           |192.168.10.212   |Worker|
|Worker2           |192.168.10.213   |Worker|
|Worker3           |192.168.10.214   |Worker|

**Control-plane node(s)**

|Protocol      |Direction |Port Range |Purpose                 |Used By              |
|--------------|----------|-----------|------------------------|---------------------|
|TCP           |Inbound   |6443*      |Kubernetes API server   |All                  |
|TCP           |Inbound   |2379-2380  |etcd server client API  |kube-apiserver, etcd |
|TCP           |Inbound   |10250      |Kubelet API             |Self, Control plane  |
|TCP           |Inbound   |10251      |kube-scheduler          |Self                 |
|TCP           |Inbound   |10252      |kube-controller-manager |Self                 |

**Worker node(s)**

|Protocol      |Direction |Port Range |Purpose                 |Used By              |
|--------------|----------|-----------|------------------------|---------------------|
|TCP           |Inbound   |10250      |Kubelet API             |Self, Control plane  |
|TCP           |Inbound   |30000-32767|NodePort Services       |All                  |

## 1. Preparing

Step1: Setup hosts trên các node

```
cat <<EOF>>/etc/hosts
192.168.10.222 master1
192.168.10.223 master2
192.168.10.224 master3
192.168.10.212 worker1
192.168.10.213 worker2
192.168.10.214 worker3
EOF
```

**Step2: Disable Selinux**

```
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

**Step3: Enable br_netfilter Kernel Module**

```
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

**Note:** Required kernel > 3.10.0.1xxx. Trong trường hợp Linux kernel thấp hơn, khi enable br_netfilter thì xảy ra lỗi

**Step4: Disable swap**

```
swapoff -a
sed -i 's/^.*swap/#&/' /etc/fstab
```

**Step5: Enable Forwarding**

```
iptables -P FORWARD ACCEPT 
cat >/etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
sysctl --system
```

**Step6: Install and configure Docker**

```
wget https://download.docker.com/linux/static/stable/x86_64/docker-18.06.3-ce.tgz
tar -zxvf docker-18.06.3-ce.tgz
cd docker
cp * /usr/local/bin
```

Hoặc cài đặt docker-ce từ repo sau:

```
curl https://download.docker.com/linux/centos/docker-ce.repo -o /etc/yum.repos.d/docker-ce.repo
yum install -y docker-ce
```

Thiết lập cgroup-driver với systemd và storage-driver với overlay2. Mặc định cgroup-driver chạy với cgroupfs và storage-driver sử dụng devicemapper.

Tạo tệp tin /etc/docker/daemon.json với nội dung sau:

```
mkdir -p /etc/docker
cat <<EOF>/etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
```

Enable Docker service and reload configuration by running the following commands

```
systemctl daemon-reload 
systemctl enable docker 
systemctl restart docker
```

Check docker information

```
[root@master1 ~]# docker info |grep -i cgroup
 Cgroup Driver: systemd
[root@master1 ~]# docker info |grep -i storage
 Storage Driver: overlay2
```

**Step7: Install kubeadm, kubectl and kubelet**

- Create repository

```
cat >/etc/yum.repos.d/kubernetes.repo<<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
- Installing kubelet, kubeadm, kubectl

`yum makecache fast && yum install -y kubelet kubeadm kubectl`

Trong trường hợp docker sử dụng cgroup là systemd, khi đó chúng ta điều chỉnh cấu hình kubelet để sử dụng systemd (mặc định cgroupfs)

```
cat <<EOF>/usr/lib/systemd/system/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/bin/kubelet --cgroup-driver=systemd
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

## 2. Setup Kubernetes cluster

### 2.1 Generating Master Configuration Files

Thiết lập cấu hình trên Master1(192.168.10.222) và sau đó copy các thông tin cấu hình đó đến các master node còn lại.

Download Cloudflare's PKI and TLS toolkit

```
curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x /usr/local/bin/cfssl*
```

Chúng ta thực hiện tạo các tệp tin json chứa thông tin để tạo self-signed certificates

```
mkdir /opt/ssl && cd /opt/ssl
cat <<EOF>ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF

cat <<EOF>ca-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "VN",
      "ST": "HN",
      "L": "HN",
      "O": "k8s",
      "OU": "system"
    }
  ]
}
EOF

cat <<EOF>etcd-csr.json 
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.10.222",
    "192.168.10.223",
    "192.168.10.224"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "VN",
      "ST": "HN",
      "L": "HN",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

Ở đây, chúng ta tạo tệp cấu hình với expire là 365 days (8760h), với các thuộc tính self-signed (CSR) tùy chọn cho cả ca và etcd

Thực hiện tạo certificate

```
cd /opt/ssl
cfssl gencert -initca ca-csr.json | cfssljson -bare ca # cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```

### 2.2 Setup ETCD Cluster

[https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)

Tạo thư mục cấu hình và copy các tệp certificate đã tạo ở bước trên đến các node master

```
mkdir -p /etc/etcd/ssl && mkdir -p /var/lib/etcd
cd /opt/ssl/
scp *.pem 192.168.10.222:/etc/etcd/ssl/
scp *.pem 192.168.10.223:/etc/etcd/ssl/
scp *.pem 192.168.10.224:/etc/etcd/ssl/
```

Trên các node master, download gói etcd (hoặc có thể cài đặt etcd từ binary package)

```
ETCD_VER=v3.4.13
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz 
tar -zxvf etcd-${ETCD_VER}-linux-amd64.tar.gz
cp etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
```

**Chạy etcd như systemd**

Thực hiện tạo tệp tin /etc/systemd/system/etcd.service trên các node master như sau:

**Trên master1**

```
cat <<EOF>/etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name master1 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.10.222:2380 \
  --listen-peer-urls https://192.168.10.222:2380 \
  --listen-client-urls https://192.168.10.222:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.10.222:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster master1=https://192.168.10.222:2380,master2=https://192.168.10.223:2380,master3=https://192.168.10.224:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**Trên master2**

```
cat <<EOF>/etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name master2 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.10.223:2380 \
  --listen-peer-urls https://192.168.10.223:2380 \
  --listen-client-urls https://192.168.10.223:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.10.223:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster master1=https://192.168.10.222:2380,master2=https://192.168.10.223:2380,master3=https://192.168.10.224:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**Trên master3**

```
cat <<EOF>/etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \
  --name master3 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://192.168.10.224:2380 \
  --listen-peer-urls https://192.168.10.224:2380 \
  --listen-client-urls https://192.168.10.224:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.10.224:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster master1=https://192.168.10.222:2380,master2=https://192.168.10.223:2380,master3=https://192.168.10.224:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

Thực hiện start etcd trên các node master

```
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
```

Show list các member trong etcd cluster

```
[root@k8s-master1 ssl]# ETCDCTL_API=3 etcdctl --write-out=table member list
+------------------+---------+---------+-----------------------------+-----------------------------+
|        ID        | STATUS  |  NAME   |         PEER ADDRS          |        CLIENT ADDRS         |
+------------------+---------+---------+-----------------------------+-----------------------------+
| 6c2d972ad2f207ae | started | master2 | https://192.168.10.223:2380 | https://192.168.10.223:2379 |
| bbdda184f3697244 | started | master3 | https://192.168.10.224:2380 | https://192.168.10.224:2379 |
| d28b4bf0112bab15 | started | master1 | https://192.168.10.222:2380 | https://192.168.10.222:2379 |
```

Note: 

- Tùy thuộc vào mô hình và số lượng member trong etcd cluster, chúng ta có thể gỡ bỏ hoặc bổ sung mới member vào etcd cluster.

- Cấu hình một số tùy chọn khác như backup/restore, upgrade, ..

## 2.3 Initial Kubernetes Cluster

### 2.3.1 On Master01

Tạo tệp tin kubeadm-config.yaml với nội dung đơn giản như sau:

```
cat <<EOF>kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
etcd:
  external:
    endpoints:
    - https://192.168.10.222:2379
    - https://192.168.10.223:2379
    - https://192.168.10.224:2379
    caFile: /etc/etcd/ssl/ca.pem
    certFile: /etc/etcd/ssl/etcd.pem
    keyFile: /etc/etcd/ssl/etcd-key.pem
networking:
  podSubnet: 10.244.0.0/16
EOF
```

Note: Trong trường sử dụng HAProxy làm LB cho Kubernetes, khi đó khai báo địa chỉ VIP làm giá trị cho *controlPlaneEndPoint*, thực hiện tạo tệp cấu hình như sau:

```
cat <<EOF>kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "192.168.10.10:6443"
etcd:
  external:
    endpoints:
    - https://192.168.10.222:2379
    - https://192.168.10.223:2379
    - https://192.168.10.224:2379
    caFile: /etc/etcd/ssl/ca.pem
    certFile: /etc/etcd/ssl/etcd.pem
    keyFile: /etc/etcd/ssl/etcd-key.pem
networking:
  podSubnet: 10.244.0.0/16
EOF
```

Thực hiện lệnh sau để khởi tạo các dịch vụ Kubernetes Master (Control-plane)

`kubeadm init --config kubeadm-config.yaml`

Quá trình khởi tạo như sau

```
[root@master1 ~]# kubeadm init --config kubeadm-config.yaml
W1030 12:02:58.484188   32057 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.19.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master1] and IPs [10.96.0.1 192.168.10.222]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] External etcd mode: Skipping etcd/ca certificate authority generation
[certs] External etcd mode: Skipping etcd/server certificate generation
[certs] External etcd mode: Skipping etcd/peer certificate generation
[certs] External etcd mode: Skipping etcd/healthcheck-client certificate generation

...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.10.222:6443 --token x22wmh.tmpbon143eyo4wql \
    --discovery-token-ca-cert-hash sha256:e58926f544f3b973ccfc16160fec68ad22b3b69649734b62267b00e125360d20 
```


Sau khi khởi tạo cluster, hệ thống sinh ra các tệp cấu hình sau:
[root@master1 ~]# ll /etc/kubernetes/manifests/
total 12
-rw------- 1 root root 3188 01:12 29 Th10 kube-apiserver.yaml
-rw------- 1 root root 2829 01:12 29 Th10 kube-controller-manager.yaml
-rw------- 1 root root 1384 01:12 29 Th10 kube-scheduler.yaml

và hệ thống lúc này tạo ra các dịch vụ sau: kube-controller, kube-scheduler, kube-proxy, kube-apiserver
Sau khi khởi tạo Kubernetes control-plane thành công, để chạy kubernetes cluster, cần thiết lập regular user như sau: 
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Thực hiện copy các tệp certificates PKI (Trừ tệp apiserver.crt and apiserver.key) đến 02 master node còn lại
```
rsync -azhP --delete --exclude=apiserver.crt --exclude=apiserver.key /etc/kubernetes/pki root@192.168.10.223:/root/kubernetes/
rsync -azhP --delete --exclude=apiserver.crt --exclude=apiserver.key /etc/kubernetes/pki root@192.168.10.224:/root/kubernetes/
```

### 2.3.2 On Master2

Tạo tệp tin kubeadm-config.yaml với nội dung đơn giản như sau:

```
cat <<EOF>kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
#apiVersion: kubeadm.k8s.io/v1
kind: ClusterConfiguration
kubernetesVersion: stable
etcd:
  external:
    endpoints:
    - https://192.168.10.222:2379
    - https://192.168.10.223:2379
    - https://192.168.10.224:2379
    caFile: /etc/etcd/ssl/ca.pem
    certFile: /etc/etcd/ssl/etcd.pem
    keyFile: /etc/etcd/ssl/etcd-key.pem
networking:
  podSubnet: 10.244.0.0/16
EOF
```

Thực hiện lệnh sau để khởi tạo các dịch vụ Kubernetes Master (Control-plane)

`kubeadm init --config kubeadm-config.yaml`

### 2.3.3 On Master3

Tạo tệp tin kubeadm-config.yaml với nội dung đơn giản như sau:

```
cat <<EOF>kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
#apiVersion: kubeadm.k8s.io/v1
kind: ClusterConfiguration
kubernetesVersion: stable
etcd:
  external:
    endpoints:
    - https://192.168.10.222:2379
    - https://192.168.10.223:2379
    - https://192.168.10.224:2379
    caFile: /etc/etcd/ssl/ca.pem
    certFile: /etc/etcd/ssl/etcd.pem
    keyFile: /etc/etcd/ssl/etcd-key.pem
networking:
  podSubnet: 10.244.0.0/16
EOF
```

Thực hiện lệnh sau để khởi tạo các dịch vụ Kubernetes Master (Control-plane)

`kubeadm init --config kubeadm-config.yaml`

