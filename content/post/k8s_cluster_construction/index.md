---
title: "K8s Cluster Construction"
subtitle: ""
description: "k8s 集群搭建"
date: 2024-12-10T02:47:58+08:00
auther: "Devin"
tags: 
    - k8s
draft: false
---

<h1 align="center">
	k8s cluster construction
</h1>


本次使用kubeadm 搭建k8s 集群，总共3台服务器（ubuntu20.04)， k8s版本为1.22.2



## 环境

- k8s-master 192.168.30.130
- k8s-node1  192.168.30.131
- k8s-node2  192.168.30.132

## 关闭防火墙

3台服务器都需要关闭防火墙

```shell
ufw disable
```

## 关闭swap

3台服务器都需要关闭swap

```shell
# 临时关闭swap
swapoff -a 

# 永久关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

## 修改主机名称

```shell
# 修改192.168.30.130 主机名称为k8s-master
hostnamectl set-hostname k8s-master

# 修改192.168.30.131 主机名称为k8s-node1
hostnamectl set-hostname k8s-node1

# 修改192.168.30.130 主机名称为k8s-node2
hostnamectl set-hostname k8s-node2
```

## 添加主机名解析

3台服务器都需要添加

```shell
vim /etc/hosts
## 添加下列文本

# k8s
192.168.30.130 k8s-master
192.168.30.131 k8s-node1
192.168.30.132 k8s-node2
```

## 安装docker

3台服务器都需要安装

```shell
cd /home
curl -fsSL https://get.docker.com -o get-docker.sh
chmod +x get-docker.sh
# 使用阿里云的镜像
./get-docker.sh --mirror Aliyun
# 等待安装成功

# 修改docker镜像加速和Cgroup Driver
vim /etc/docker/daemon.json
# 修改为
{
	"registry-mirrors": ["https://dhq9bx4f.mirror.aliyuncs.com"],
	"exec-opts": ["native.cgroupdriver=systemd"]
}

# 重启docker 
systemctl daemon-reload
systemctl restart docker
```

## 安装 kubelet kubectl kubeadm

3台服务器都需要安装

```shell
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
#新增源
add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"

apt-get update
# 查看是否存在该版本
apt-cache madison kubelet kubectl kubeadm | grep '1.22.2-00' 
apt-get install -y kubelet=1.22.2-00 kubectl=1.22.2-00 kubeadm=1.22.2-00 

## 等待安装成功
kubectl version --client=true -o json
{
  "clientVersion": {
    "major": "1",
    "minor": "22",
    "gitVersion": "v1.22.2",
    "gitCommit": "8b5a19147530eaac9476b0ab82980b4088bbc1b2",
    "gitTreeState": "clean",
    "buildDate": "2021-09-15T21:38:50Z",
    "goVersion": "go1.16.8",
    "compiler": "gc",
    "platform": "linux/amd64"
  }
}

# 有以上显示 表示安装成功
```

## 初始化k8s-master

```shell
# apiserver-advertise-address 代表你的k8s-master的ip
# image-repository 使用阿里云的镜像
kubeadm init \
--apiserver-advertise-address=192.168.30.130 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.22.2 \
--ignore-preflight-errors=Swap \
--pod-network-cidr=10.244.0.0/16 \
--service-cidr=10.1.0.0/16 

## 等待安装，成功后执行
# start using your cluster, you need to run the following as a regular user:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Alternatively, if you are the root user, you can run:
export KUBECONFIG=/etc/kubernetes/admin.conf
```

```text/plain
# 安装成功后

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.30.130:6443 --token 56bthi.poohqb2p2mavk1am \
        --discovery-token-ca-cert-hash sha256:c964563c7a38a633746225ff6d23f40fde626d7d796b01bffb299a84efbdcf82
```

## k8s网络

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at: https://kubernetes.io/docs/concepts/cluster-administration/addons/

使用的[flannel](https://github.com/coreos/flannel)的overlay 实现多节点pod通信

```shell
# k8s-master 
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# 成功后使用kubectl 查看pods
kubectl get pods -A
kube-system   coredns-7f6cbbb7b8-ff64c             1/1     Running   0          23h
kube-system   coredns-7f6cbbb7b8-txc5w             1/1     Running   0          23h
kube-system   etcd-k8s-master                      1/1     Running   0          23h
kube-system   kube-apiserver-k8s-master            1/1     Running   0          23h
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          23h
kube-system   kube-flannel-ds-8g5st                1/1     Running   0          23h
kube-system   kube-flannel-ds-njjgg                1/1     Running   0          23h
kube-system   kube-flannel-ds-v46xq                1/1     Running   0          22h
kube-system   kube-proxy-l7rxg                     1/1     Running   0          23h
kube-system   kube-proxy-pfcbb                     1/1     Running   0          23h
kube-system   kube-proxy-q289f                     1/1     Running   0          22h
kube-system   kube-scheduler-k8s-master            1/1     Running   0          23h
```

## k8s-node 加入集群

```shell
# 复制k8s-master的输出信息
kubeadm join 192.168.30.130:6443 --token 56bthi.poohqb2p2mavk1am \
        --discovery-token-ca-cert-hash sha256:c964563c7a38a633746225ff6d23f40fde626d7d796b01bffb299a84efbdcf82
# 等待加入成功
kubectl get nodes
k8s-master   Ready    control-plane,master   23h   v1.22.2
k8s-node1    Ready    <none>                 23h   v1.22.2
k8s-node2    Ready    <none>                 23h   v1.22.2
```

## 测试k8s集群

```shell
# 创建nginx容器
kubectl create deployment nginx --image=nginx
# 暴露对外端口
kubectl expose deployment nginx --port=80 --type=NodePort
# 扩容副本
kubectl scale deployment nginx --replicas=3
# 查看nginx是否运行成功
kubectl get pod,svc
NAME                         READY   STATUS    RESTARTS   AGE
pod/nginx-6799fc88d8-2bps9   1/1     Running   0          23h
pod/nginx-6799fc88d8-9psfx   1/1     Running   0          22h
pod/nginx-6799fc88d8-tnfrk   1/1     Running   0          22h

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        23h
service/nginx        NodePort    10.102.105.92   <none>        80:31150/TCP   23h
root@k8s-master:~#
# 浏览器访问 http://192.168.30.130:31150 http://192.168.30.131:31150  http://192.168.30.132:31150  均可以成功访问
```
