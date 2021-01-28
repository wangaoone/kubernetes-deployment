## Kubernetes setup

### 安装container runtime

我们选择`docker`作为container runtime, 安装环境为`centos 7`

```shell
# (Install Docker CE)
## Set up the repository
### Install required packages
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

```shell
## Add the Docker repository
sudo yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
```

```shell
# Install Docker CE
sudo yum update -y && sudo yum install -y \
  containerd.io-1.2.13 \
  docker-ce-19.03.11 \
  docker-ce-cli-19.03.11
```

```shell
## Create /etc/docker
sudo mkdir /etc/docker
```

```shell
# Set up the Docker daemon
cat <<EOF | sudo tee /etc/docker/daemon.json
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

```shell
# Create /etc/systemd/system/docker.service.d
sudo mkdir -p /etc/systemd/system/docker.service.d
```

```shell
# Restart Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### Kubeadm

#### 配置kubeadm

```shell
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

#### 安装kubeadm, kubelet and kubectl

- kubeadm: command to boostrap the cluster
- kubelet: 在所有机器上的组件，启动pods和containers
- kubectl: cmd line tool

```shell
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

repo文件可能因为代理问题访问不到，refer to this [link](https://developer.aliyun.com/mirror/kubernetes)

重启kubelet

```shell
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

#### kubeadm init & create cluster

- 初始化kubeadm

```shell
[root@iZm5e8d2tp1boi0kol074mZ ~]# kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.20.2
k8s.gcr.io/kube-controller-manager:v1.20.2
k8s.gcr.io/kube-scheduler:v1.20.2
k8s.gcr.io/kube-proxy:v1.20.2
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.13-0
k8s.gcr.io/coredns:1.7.0
```

因为镜像代理问题，需要手动下载镜像并重新`tag`后再执行`init`
脚本：其中版本号需要对应上面的版本号

```shell
# vim kubeadm.sh

#!/bin/bash

## 使用如下脚本下载国内镜像，并修改tag为google的tag
set -e

KUBE_VERSION=v1.20.2
KUBE_PAUSE_VERSION=3.2
ETCD_VERSION=3.4.13-0
CORE_DNS_VERSION=1.7.0

GCR_URL=k8s.gcr.io
ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

images=(kube-proxy:${KUBE_VERSION}
kube-scheduler:${KUBE_VERSION}
kube-controller-manager:${KUBE_VERSION}
kube-apiserver:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION}
etcd:${ETCD_VERSION}
coredns:${CORE_DNS_VERSION})

for imageName in ${images[@]} ; do
  docker pull $ALIYUN_URL/$imageName
  docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
  docker rmi $ALIYUN_URL/$imageName
done
```

- master节点配置

```shell
sudo kubeadm init \
# --apiserver-advertise-address 192.168.10.20 \
 --pod-network-cidr=10.244.0.0/16
```

note: `--pod-network-cidr`暂时不要修改

- worker 节点配置

执行节点加入操作，参考上一步kubeadm init的输出内容

- 安装flanneld

```shell
sudo mkdir ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/

cd ~/.kube

sudo mv admin.conf config
sudo service kubelet restart
```

```shell
kubectl apply -f http://zabbix.itunesapplestore.com.cn/ray/k8s-sh/kube-flannel.yml
```

如果出现错误，将可用的镜像复制worker镜像启动即可