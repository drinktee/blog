---
title: Kubernetes 1.6 集群部署
date: 2017-05-20 18:27:01
tags:
---

# Kubernetes 1.6 集群部署

# 环境准备

- 需要先升级内核到3.10以上

### 下载安装包

```bash
wget https://dl.k8s.io/v1.6.2/kubernetes-server-linux-amd64.tar.gz
tar -zxvf kubernetes-server-linux-amd64.tar.gz
```

### 系统配置

在各节点创建/etc/sysctl.d/k8s.conf文件，添加如下内容：

```bash
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

执行命令使修改生效

```bash
setenforce 0
vi /etc/selinux/config
SELINUX=disabled
```

### ETCD集群，Docker安装

etcd集群采用高可用，三节点部署，启用TLS。在每个node上安装好docker。

# Kubernetes各组件TLS证书和密钥生成

使用[CFSSL](https://github.com/cloudflare/cfssl)来生成证书和密钥。

### 生成CA证书和私钥

`ca-config.json`文件

```json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "frognew": {
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
```

创建CA证书签名请求配置ca-csr.json：

```json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "cloudnative"
    }
  ]
}
```

使用cfssl生成CA证书和私钥

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

### 生成kube-apiserver证书和私钥

有三个master节点，"11.1.0.1"是api-server的service ip
apiserver-csr.json：

```bash
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "10.73.212.42",
      "10.73.213.14",
      "10.73.213.31",
      "11.1.0.1",
      "master-mgr00",
      "master-mgr01",
      "master-mgr02",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "cloudnative"
        }
    ]
}
```

然后生成kube-apiserver的证书和私钥：

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=frognew apiserver-csr.json | cfssljson -bare apiserver
ls apiserver*
apiserver.csr  apiserver-csr.json  apiserver-key.pem  apiserver.pem
```

### 创建kubernetes-admin客户端证书和私钥

```json
{
  "CN": "kubernetes-admin",
  "hosts": [
      "10.73.212.42",
      "10.73.213.14",
      "10.73.213.31",
      "master-mgr00",
      "master-mgr01",
      "master-mgr02"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "cloudnative"
    }
  ]
}
```

生成kubernetes-admin的证书和私钥：

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=frognew admin-csr.json | cfssljson -bare admin
```

### 生成kube-controller-manager证书和私钥

```json
{
  "CN": "system:kube-controller-manager",
  "hosts": [
      "10.73.212.42",
      "10.73.213.14",
      "10.73.213.31",
      "master-mgr00",
      "master-mgr01",
      "master-mgr02"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:kube-controller-manager",
      "OU": "cloudnative"
    }
  ]
}
```

生成证书和私钥

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=frognew controller-manager-csr.json | cfssljson -bare controller-manager
```

### 生成kube-scheduler证书和私钥

scheduler-csr.json：

```json
{
  "CN": "system:kube-scheduler",
  "hosts": [
      "10.73.212.42",
      "10.73.213.14",
      "10.73.213.31",
      "master-mgr00",
      "master-mgr01",
      "master-mgr02"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:kube-scheduler",
      "OU": "cloudnative"
    }
  ]
}
```

命令仍然类似

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=frognew scheduler-csr.json | cfssljson -bare scheduler
```

# 部署三个master节点

三个节点需要先拷贝好各组件的证书与密钥，参考的组件启动参数如下。

*注意：这里没有做高可用！*

### kube-apiserver

```bash
/home/work/kube_apiserver/bin/kube-apiserver \
--bind-address=0.0.0.0 --insecure-port=8080 \
--secure-port=8443  \
--etcd_servers=https://master-mgr00:2379,\
https://master-mgr01:2379, \
https://master-mgr02:2379 \
--apiserver-count=3 --logtostderr=true --allow-privileged=true \
--tls-cert-file=/home/work/kube_apiserver/conf/apiserver.pem \
--tls-private-key-file=/home/work/kube_apiserver/conf/apiserver-key.pem \
--client-ca-file=/home/work/kube_apiserver/conf/ca.pem \
--service-account-key-file=/home/work/kube_apiserver/conf/ca-key.pem \
--etcd-cafile=/home/work/kube_apiserver/conf/etcd/ca.pem \
--etcd-certfile=/home/work/kube_apiserver/conf/etcd/etcd.pem \
--etcd-keyfile=/home/work/kube_apiserver/conf/etcd/etcd-key.pem \
--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,\
PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds \
--authorization-mode=RBAC --service-cluster-ip-range=11.1.0.0/16
```

### kube-controller-manager

```bash
/home/work/kube_controller_manager/bin/kube-controller-manager \
--address=0.0.0.0 --master=http://127.0.0.1:8080 \
--leader-elect=true --logtostderr=true \
 --cluster-cidr=172.17.0.0/16 --allocate-node-cidrs=true \
 --service-cluster-ip-range=11.1.0.0/16 \
 --service-account-private-key-file=/home/work/kube_controller_manager/conf/ca-key.pem \
 --root-ca-file=/home/work/kube_controller_manager/conf/ca.pem
```

### kube-scheduler

```bash
/home/work/kube_scheduler/bin/kube-scheduler \
--address=0.0.0.0 --leader-elect=true --logtostderr=true \
--master=http://127.0.0.1:8080
```

# Kubernetes Node节点部署

- 部分机器要先安装下载`nsenter`这个二进制工具到bin目录

### CNI安装

```bash
wget https://github.com/containernetworking/cni/releases/download/v0.5.2/cni-amd64-v0.5.2.tgz
mkdir -p /opt/cni/bin
tar -zxvf cni-amd64-v0.5.2.tgz -C /opt/cni/bin
ls /opt/cni/bin/
bridge  cnitool  dhcp  flannel  host-local  ipvlan  loopback  macvlan  noop  ptp  tuning
```

### kubelet

kubelet-csr.json：

```json
{
  "CN": "system:node",
  "hosts": [
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:nodes",
      "OU": "cloudnative"
    }
  ]
}
```

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=frognew kubelet-csr.json | cfssljson -bare kubelet
```

启动参数

```bash
 /home/work/kubelet/bin/kubelet --address=0.0.0.0 \
 --api-servers=https://master-mgr00:8443,https://master-mgr00:8443,https://master-mgr00:8443 \
 --pod_infra_container_image=registry.baidu.com/public/pause:2.0 \
 --cluster-domain=cluster.local --cluster-dns=11.1.0.10 \
 --logtostderr=true \
  --client-ca-file=/home/work/kubelet/conf/ca.pem \
  --tls-private-key-file=/home/work/kubelet/conf/kubelet-key.pem \
  --tls-cert-file=/home/work/kubelet/conf/kubelet.pem \
  --kubeconfig=/home/work/kubelet/conf/kubeconfig.yaml \
  --allow-privileged=true --cgroups-per-qos=false \
  --enforce-node-allocatable= --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin \
  --cadvisor-port=8086
```


### kube-proxy

kube-proxy-csr.json:

```json
{
  "CN": "system:kube-proxy",
  "hosts": [
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:kube-proxy",
      "OU": "cloudnative"
    }
  ]
}
```

```bash
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=frognew kube-proxy-csr.json | cfssljson -bare kube-proxy
```

生成kubeconfig：

```bash
export KUBE_APISERVER="https://master-mgr00:8443"
# set-cluster
kubectl config set-cluster kubernetes \
  --certificate-authority=/home/work/kube_proxy/conf/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.conf
# set-credentials
kubectl config set-credentials system:kube-proxy \
  --client-certificate=/home/work/kube_proxy/conf/kube-proxy.pem \
  --embed-certs=true \
  --client-key=/home/work/kube_proxy/conf/kube-proxy-key.pem \
  --kubeconfig=kube-proxy.conf
# set-context
kubectl config set-context system:kube-proxy@kubernetes \
  --cluster=kubernetes \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.conf
# set default context
kubectl config use-context system:kube-proxy@kubernetes --kubeconfig=kube-proxy.conf
```

启动命令

```bash
 /home/work/kube_proxy/bin/kube-proxy --master=https://master-mgr00:8443 \
 --kubeconfig=/home/work/kube_proxy/conf/kube-proxy.conf \
 --proxy-mode=iptables --cluster-cidr=172.17.0.0/16 \
 --logtostderr=true
```

# 总结

后续还要部署flannel，dns等其他插件，将写在下一篇博客里了。