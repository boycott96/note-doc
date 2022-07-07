---
sidebar_position: 2
---

# 二进制方式安装
我们下面的安装方式就是单纯的使用二进制方式安装，并没有对 Kube-APIServer 组件进行高可用配置，因为像我们安装 K8s 的话，其实主要还是为了学习 K8s，通过 K8s 来完成某些事情，所以并不需要关心高可用这块的东西。

要是对Kubernetes 做高可用的话，其实并不难，像一些在云上的 K8s，一般都是通过 SLB 来代理到两台不同服务器上，来实现高可用；而像云下的 K8s，基本上也是如上，我们可以通过 Keepalived 加 Nginx 来实现高可用。

准备工作：

| 主机名         | 操作系统   | IP 地址     | 所需组件                            |
| :------------- | :--------- | :---------- | :---------------------------------- |
| `k8s-master01` | CentOS 7.4 | 192.168.1.1 | 所有组件都安装 **（合理利用资源）** |
| `k8s-master02` | CentOS 7.4 | 192.168.1.2 | 所有组件都安装                      |
| `k8s-node`     | CentOS 7.4 | 192.168.1.3 | `docker` `kubelet` `kube-proxy`     |

1）在各个节点上配置主机名，并配置 Hosts 文件

```
[root@ddkk.com ~]# hostnamectl set-hostname k8s-master01
[root@ddkk.com ~]# bash
[root@k8s-master01 ~]# cat <<END >> /etc/hosts
192.168.1.1 k8s-master01
192.168.1.2 k8s-master02
192.168.1.3 k8s-node01
END
```

2）在`k8s-master01` 上配置 SSH 密钥对，并将公钥发送给其余主机

```
[root@k8s-master01 ~]# ssh-keygen -t rsa            # 三连回车
[root@k8s-master01 ~]# ssh-copy-id root@192.168.1.1
[root@k8s-master01 ~]# ssh-copy-id root@192.168.1.2
[root@k8s-master01 ~]# ssh-copy-id root@192.168.1.3
```

3）编写 K8s 初始环境脚本

```
[root@k8s-master01 ~]# vim k8s-init.sh
#!/bin/bash
#****************************************************************#
# ScriptName: k8s-init.sh
# Initialize the machine. This needs to be executed on every machine.
# Mkdir k8s directory
yum -y install wget ntpdate && ntpdate ntp1.aliyun.com
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
yum -y install epel-release
mkdir -p /opt/k8s/bin/
mkdir -p /data/k8s/docker
mkdir -p /data/k8s/k8s
# Disable the SELinux.
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
# Turn off and disable the firewalld.
systemctl stop firewalld
systemctl disable firewalld
# Modify related kernel parameters & Disable the swap.
cat > /etc/sysctl.d/k8s.conf << EOF
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.tcp_tw_recycle = 0
vm.swappiness = 0
vm.overcommit_memory = 1
vm.panic_on_oom = 0
net.ipv6.conf.all.disable_ipv6 = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf >& /dev/null
# Add ipvs modules
cat > /etc/sysconfig/modules/ipvs.modules << EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- br_netfilter
modprobe -- nf_conntrack
modprobe -- nf_conntrack_ipv4
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules
source /etc/sysconfig/modules/ipvs.modules
# Install rpm
yum install -y conntrack ntpdate ntp ipvsadm ipset jq iptables curl sysstat libseccomp wget gcc gcc-c++ make libnl libnl-devel libnfnetlink-devel openssl-devel vim openssl-devel bash-completion
# ADD k8s bin to PATH
echo 'export PATH=/opt/k8s/bin:$PATH' >> /root/.bashrc && chmod +x /root/.bashrc && source /root/.bashrc
[root@k8s-master01 ~]# bash k8s-init.sh
```

4）配置环境变量

```
[root@k8s-master01 ~]# vim environment.sh
#!/bin/bash
# 生成 EncryptionConfig 所需的加密 Key
export ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

# 集群 Master 机器 IP 数组
export MASTER_IPS=(192.168.1.1 192.168.1.2)

# 集群 Master IP 对应的主机名数组
export MASTER_NAMES=(k8s-master01 k8s-master02)

# 集群 Node 机器 IP 数组
export NODE_IPS=(192.168.1.3)

# 集群 Node IP 对应的主机名数组
export NODE_NAMES=(k8s-node01)

# 集群所有机器 IP 数组
export ALL_IPS=(192.168.1.1 192.168.1.2 192.168.1.3)

# 集群所有 IP 对应的主机名数组
export ALL_NAMES=(k8s-master01 k8s-master02 k8s-node01)

# Etcd 集群服务地址列表
export ETCD_ENDPOINTS="https://192.168.1.1:2379,https://192.168.1.2:2379"

# Etcd 集群间通信的 IP 和端口
export ETCD_NODES="k8s-master01=https://192.168.1.1:2380,k8s-master02=https://192.168.1.2:2380"

# Kube-apiserver 的 IP 和端口
export KUBE_APISERVER="https://192.168.1.1:6443"

# 节点间互联网络接口名称
export IFACE="ens32"

# Etcd 数据目录
export ETCD_DATA_DIR="/data/k8s/etcd/data"

# Etcd WAL 目录. 建议是 SSD 磁盘分区. 或者和 ETCD_DATA_DIR 不同的磁盘分区
export ETCD_WAL_DIR="/data/k8s/etcd/wal"

# K8s 各组件数据目录
export K8S_DIR="/data/k8s/k8s"

# Docker 数据目录
export DOCKER_DIR="/data/k8s/docker"

## 以下参数一般不需要修改
# TLS Bootstrapping 使用的 Token. 可以使用命令 head -c 16 /dev/urandom | od -An -t x | tr -d ' ' 生成
BOOTSTRAP_TOKEN="41f7e4ba8b7be874fcff18bf5cf41a7c"

# 最好使用当前未用的网段来定义服务网段和 Pod 网段
# 服务网段. 部署前路由不可达. 部署后集群内路由可达(kube-proxy 保证)
SERVICE_CIDR="10.20.0.0/16"

# Pod 网段. 建议 /16 段地址. 部署前路由不可达. 部署后集群内路由可达(flanneld 保证)
CLUSTER_CIDR="10.10.0.0/16"

# 服务端口范围 (NodePort Range)
export NODE_PORT_RANGE="1-65535"

# Flanneld 网络配置前缀
export FLANNEL_ETCD_PREFIX="/kubernetes/network"

# Kubernetes 服务 IP (一般是 SERVICE_CIDR 中第一个 IP)
export CLUSTER_KUBERNETES_SVC_IP="10.20.0.1"

# 集群 DNS 服务 IP (从 SERVICE_CIDR 中预分配)
export CLUSTER_DNS_SVC_IP="10.20.0.254"

# 集群 DNS 域名(末尾不带点号)
export CLUSTER_DNS_DOMAIN="cluster.local"

# 将二进制目录 /opt/k8s/bin 加到 PATH 中
export PATH=/opt/k8s/bin:$PATH
```

- 上面像那些 IP 地址和网卡啥的，你们要改成自身对应的信息。

```
[root@k8s-master01 ~]# chmod +x environment.sh && source environment.sh
```

下面的这些操作，我们只需要在 `k8s-master01` 主机上操作即可（因为下面我们会通过 `for` 循环来发送到其余主机上）

## 1.创建 CA 证书和密钥

因为Kubernetes 系统的各个组件需要使用 TLS 证书对其通信加密以及授权认证，所以我们需要在安装前先生成相关的 TLS 证书；我们可以使用 `openssl` `cfssl` `easyrsa` 来生成 Kubernetes 的相关证书，我们下面使用的是 `cfssl` 方式。

1）安装 `cfssl` 工具集

```
[root@k8s-master01 ~]# mkdir -p /opt/k8s/cert
[root@k8s-master01 ~]# curl -L https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /opt/k8s/bin/cfssl
[root@k8s-master01 ~]# curl -L https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /opt/k8s/bin/cfssljson
[root@k8s-master01 ~]# curl -L https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o /opt/k8s/bin/cfssl-certinfo
[root@k8s-master01 ~]# chmod +x /opt/k8s/bin/*
```

2）创建根证书配置文件

```
[root@k8s-master01 ~]# mkdir -p /opt/k8s/work
[root@k8s-master01 ~]# cd /opt/k8s/work/
[root@k8s-master01 work]# cat > ca-config.json << EOF
{
   
     
    "signing": {
   
     
        "default": {
   
     
            "expiry": "876000h"
        },
        "profiles": {
   
     
            "kubernetes": {
   
     
                "expiry": "876000h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
```

- signing：表示当前证书可用于签名其它证书；
- server auth：表示 Client 可以用这个 CA 对 Server 提供的证书进行校验；
- client auth：表示 Server 可以用这个 CA 对 Client 提供的证书进行验证；
- "expiry": "876000h"：表示当前证书有效期为 100 年；

3）创建根证书签名请求文件

```
[root@k8s-master01 work]# cat > ca-csr.json << EOF
{
   
     
    "CN": "kubernetes",
    "key": {
   
     
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
   
     
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "k8s",
            "OU": "System"
        }
    ],
    "ca": {
   
     
        "expiry": "876000h"
 }
}
EOF
```

- CN：Kube-APIServer 将会把这个字段作为请求的用户名，来让浏览器验证网站是否合法。
- C：**国家**；ST：**州，省**；L：**地区，城市**；O：**组织名称，公司名称**；OU：**组织单位名称，公司部门。**

4）生成 CA 密钥 `ca-key.pem` 和证书 `ca.pem`

```
[root@k8s-master01 work]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

- 生成证书后，因为 Kubernetes 集群需要 **双向 TLS 认证**，所以我们可以将生成的文件传送到所有主机中。

5）使用 `for` 循环来遍历数组，将配置发送给所有主机

```
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    ssh root@${
   
     all_ip} "mkdir -p /etc/kubernetes/cert"
    scp ca*.pem ca-config.json root@${
   
     all_ip}:/etc/kubernetes/cert
  done
```

## 2.安装 ETCD 组件

ETCD 是基于 Raft 的分布式 `key-value` 存储系统，由 CoreOS 开发，常用于服务发现、共享配置以及并发控制（如 `leader`选举、分布式锁等）；Kubernetes 主要就是用 ETCD 来存储所有的运行数据。

下载ETCD

```
[root@k8s-master01 work]# wget https://github.com/etcd-io/etcd/releases/download/v3.3.22/etcd-v3.3.22-linux-amd64.tar.gz
[root@k8s-master01 work]# tar -zxf etcd-v3.3.22-linux-amd64.tar.gz
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp etcd-v3.3.22-linux-amd64/etcd* root@${
   
     master_ip}:/opt/k8s/bin
    ssh root@${
   
     master_ip} "chmod +x /opt/k8s/bin/*"
  done
```

### 1）创建 ETCD 证书和密钥

```
[root@k8s-master01 work]# cat > etcd-csr.json << EOF
{
   
     
    "CN": "etcd",
    "hosts": [
        "127.0.0.1",
        "192.168.1.1",
        "192.168.1.2"
  ],
    "key": {
   
     
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
   
     
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

- hosts：用来指定给 ETCD 授权的 IP 地址或域名列表。

### 2）生成证书和密钥

```
[root@k8s-master01 work]# cfssl gencert -ca=/opt/k8s/work/ca.pem -ca-key=/opt/k8s/work/ca-key.pem -config=/opt/k8s/work/ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    ssh root@${
   
     master_ip} "mkdir -p /etc/etcd/cert"
    scp etcd*.pem root@${
   
     master_ip}:/etc/etcd/cert/
  done
```

### 3）创建启动脚本

```
[root@k8s-master01 work]# cat > etcd.service.template << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=${
   
     ETCD_DATA_DIR}
ExecStart=/opt/k8s/bin/etcd \\
  --enable-v2=true \\
  --data-dir=${
   
     ETCD_DATA_DIR} \\
  --wal-dir=${
   
     ETCD_WAL_DIR} \\
  --name=##MASTER_NAME## \\
  --cert-file=/etc/etcd/cert/etcd.pem \\
  --key-file=/etc/etcd/cert/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-cert-file=/etc/etcd/cert/etcd.pem \\
  --peer-key-file=/etc/etcd/cert/etcd-key.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls=https://##MASTER_IP##:2380 \\
  --initial-advertise-peer-urls=https://##MASTER_IP##:2380 \\
  --listen-client-urls=https://##MASTER_IP##:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://##MASTER_IP##:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${
   
     ETCD_NODES} \\
  --initial-cluster-state=new \\
  --auto-compaction-mode=periodic \\
  --auto-compaction-retention=1 \\
  --max-request-bytes=33554432 \\
  --quota-backend-bytes=6442450944 \\
  --heartbeat-interval=250 \\
  --election-timeout=2000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
[root@k8s-master01 work]# for (( A=0; A < 2; A++ ))
do
sed -e "s/##MASTER_NAME##/${MASTER_NAMES[A]}/" -e "s/##MASTER_IP##/${MASTER_IPS[A]}/" etcd.service.template > etcd-${
   
     MASTER_IPS[A]}.service
done
```

### 4）启动 ETCD

```
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp etcd-${
   
     master_ip}.service root@${
   
     master_ip}:/etc/systemd/system/etcd.service
    ssh root@${
   
     master_ip} "mkdir -p ${ETCD_DATA_DIR} ${ETCD_WAL_DIR}"
    ssh root@${
   
     master_ip} "systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd"
  done
```

查看ETCD 当前的 Leader（领导）

```
[root@k8s-master01 work]# ETCDCTL_API=3 /opt/k8s/bin/etcdctl \
-w table --cacert=/etc/kubernetes/cert/ca.pem \
--cert=/etc/etcd/cert/etcd.pem \
--key=/etc/etcd/cert/etcd-key.pem \
--endpoints=${
   
     ETCD_ENDPOINTS} endpoint status
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==) 

## 3.安装 Flannel 网络插件

Flannel 是一种基于 `overlay` 网络的跨主机容器网络解决方案，也就是将 TCP 数据封装在另一种网络包里面进行路由转发和通信。Flannel 是使用 Go 语言开发的，主要就是用来让不同主机内的容器实现互联。

下载Flannel

```
[root@k8s-master01 work]# mkdir flannel
[root@k8s-master01 work]# wget https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz
[root@k8s-master01 work]# tar -zxf flannel-v0.11.0-linux-amd64.tar.gz -C flannel
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    scp flannel/{
   
     flanneld,mk-docker-opts.sh} root@${
   
     all_ip}:/opt/k8s/bin/
    ssh root@${
   
     all_ip} "chmod +x /opt/k8s/bin/*"
  done
```

### 1）创建 Flannel 证书和密钥

```
[root@k8s-master01 work]# cat > flanneld-csr.json << EOF
{
   
     
    "CN": "flanneld",
    "hosts": [],
    "key": {
   
     
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
   
     
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

### 2）生成证书和密钥

```
[root@k8s-master01 work]# cfssl gencert -ca=/opt/k8s/work/ca.pem -ca-key=/opt/k8s/work/ca-key.pem -config=/opt/k8s/work/ca-config.json -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    ssh root@${
   
     all_ip} "mkdir -p /etc/flanneld/cert"
    scp flanneld*.pem root@${
   
     all_ip}:/etc/flanneld/cert
  done
```

配置Pod 的网段信息

```
[root@k8s-master01 work]# etcdctl \
--endpoints=${
   
     ETCD_ENDPOINTS} \
--ca-file=/opt/k8s/work/ca.pem \
--cert-file=/opt/k8s/work/flanneld.pem \
--key-file=/opt/k8s/work/flanneld-key.pem \
mk ${
   
     FLANNEL_ETCD_PREFIX}/config '{"Network":"'${
   
     CLUSTER_CIDR}'", "SubnetLen": 21, "Backend": {"Type": "vxlan"}}'
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==) 

### 3）编写启动脚本

```
[root@k8s-master01 work]# cat > flanneld.service << EOF
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/opt/k8s/bin/flanneld \\
  -etcd-cafile=/etc/kubernetes/cert/ca.pem \\
  -etcd-certfile=/etc/flanneld/cert/flanneld.pem \\
  -etcd-keyfile=/etc/flanneld/cert/flanneld-key.pem \\
  -etcd-endpoints=${
   
     ETCD_ENDPOINTS} \\
  -etcd-prefix=${
   
     FLANNEL_ETCD_PREFIX} \\
  -iface=${
   
     IFACE} \\
  -ip-masq
ExecStartPost=/opt/k8s/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=always
RestartSec=5
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
```

### 4）启动并验证

```
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    scp flanneld.service root@${
   
     all_ip}:/etc/systemd/system/
    ssh root@${
   
     all_ip} "systemctl daemon-reload && systemctl enable flanneld --now"
  done
```

1）查看 Pod 网段信息

```
[root@k8s-master01 work]# etcdctl \
--endpoints=${
   
     ETCD_ENDPOINTS} \
--ca-file=/etc/kubernetes/cert/ca.pem \
--cert-file=/etc/flanneld/cert/flanneld.pem \
--key-file=/etc/flanneld/cert/flanneld-key.pem \
get ${
   
     FLANNEL_ETCD_PREFIX}/config
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)
2）查看已分配的 Pod 子网段列表

```
[root@k8s-master01 work]# etcdctl \
--endpoints=${
   
     ETCD_ENDPOINTS} \
--ca-file=/etc/kubernetes/cert/ca.pem \
--cert-file=/etc/flanneld/cert/flanneld.pem \
--key-file=/etc/flanneld/cert/flanneld-key.pem \
ls ${
   
     FLANNEL_ETCD_PREFIX}/subnets
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)
3）查看某一 Pod 网段对应的节点 IP 和 Flannel 接口地址

```
[root@k8s-master01 work]# etcdctl \
--endpoints=${
   
     ETCD_ENDPOINTS} \
--ca-file=/etc/kubernetes/cert/ca.pem \
--cert-file=/etc/flanneld/cert/flanneld.pem \
--key-file=/etc/flanneld/cert/flanneld-key.pem \
get ${
   
     FLANNEL_ETCD_PREFIX}/subnets/10.10.208.0-21
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==) 

## 4.安装 Docker 服务

Docker 运行和管理容器，Kubelet 通过 Container Runtime Interface (CRI) 与它进行交互。

下载Docker

```
[root@k8s-master01 work]# wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.12.tgz
[root@k8s-master01 work]# tar -zxf docker-19.03.12.tgz
```

安装Docker

```
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    scp docker/*  root@${
   
     all_ip}:/opt/k8s/bin/
    ssh root@${
   
     all_ip} "chmod +x /opt/k8s/bin/*"
  done
```

### 1）创建启动脚本

```
[root@k8s-master01 work]# cat > docker.service << "EOF"
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
WorkingDirectory=##DOCKER_DIR##
Environment="PATH=/opt/k8s/bin:/bin:/sbin:/usr/bin:/usr/sbin"
EnvironmentFile=-/run/flannel/docker
ExecStart=/opt/k8s/bin/dockerd $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
[root@k8s-master01 work]# sed -i -e "s|##DOCKER_DIR##|${DOCKER_DIR}|" docker.service
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    scp docker.service root@${
   
     all_ip}:/etc/systemd/system/
  done
```

配置`daemon.json` 文件

```
[root@k8s-master01 work]# cat > daemon.json << EOF
{
   
     
    "registry-mirrors": ["https://ipbtg5l0.mirror.aliyuncs.com"],
    "exec-opts": ["native.cgroupdriver=cgroupfs"],
    "data-root": "${DOCKER_DIR}/data",
    "exec-root": "${DOCKER_DIR}/exec",
    "log-driver": "json-file",
    "log-opts": {
   
     
      "max-size": "100m",
      "max-file": "5"
    },
    "storage-driver": "overlay2",
    "storage-opts": [
      "overlay2.override_kernel_check=true"
  ]
}
EOF
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    ssh root@${
   
     all_ip} "mkdir -p /etc/docker/ ${DOCKER_DIR}/{data,exec}"
    scp daemon.json root@${
   
     all_ip}:/etc/docker/daemon.json
  done
```

### 2）启动 Docker

```
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    ssh root@${
   
     all_ip} "systemctl daemon-reload && systemctl enable docker --now"
  done
```

## 5.安装 Kubectl 服务

下载Kubectl

```
[root@k8s-master01 work]# wget https://storage.googleapis.com/kubernetes-release/release/v1.18.3/kubernetes-client-linux-amd64.tar.gz
[root@k8s-master01 work]# tar -zxf kubernetes-client-linux-amd64.tar.gz
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp kubernetes/client/bin/kubectl root@${
   
     master_ip}:/opt/k8s/bin/
    ssh root@${
   
     master_ip} "chmod +x /opt/k8s/bin/*"
  done
```

### 1）创建 Admin 证书和密钥

```
[root@k8s-master01 work]# cat > admin-csr.json << EOF
{
   
     
    "CN": "admin",
    "hosts": [],
    "key": {
   
     
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
   
     
            "C": "CN",
            "ST": "Shanghai",
            "L": "Shanghai",
            "O": "system:masters",
            "OU": "System"
        }
    ]
}
EOF
```

### 3）生成证书和密钥

```
[root@k8s-master01 work]# cfssl gencert -ca=/opt/k8s/work/ca.pem -ca-key=/opt/k8s/work/ca-key.pem -config=/opt/k8s/work/ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

### 4）创建 Kubeconfig 文件

配置集群参数

```
[root@k8s-master01 work]# kubectl config set-cluster kubernetes \
--certificate-authority=/opt/k8s/work/ca.pem \
--embed-certs=true \
--server=${
   
     KUBE_APISERVER} \
--kubeconfig=kubectl.kubeconfig
```

配置客户端认证参数

```
[root@k8s-master01 work]# kubectl config set-credentials admin \
--client-certificate=/opt/k8s/work/admin.pem \
--client-key=/opt/k8s/work/admin-key.pem \
--embed-certs=true \
--kubeconfig=kubectl.kubeconfig
```

配置上下文参数

```
[root@k8s-master01 work]# kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=admin \
--kubeconfig=kubectl.kubeconfig
```

配置默认上下文

```
[root@k8s-master01 work]# kubectl config use-context kubernetes --kubeconfig=kubectl.kubeconfig
```

### 5）创建 Kubectl 配置文件，并配置命令补全工具

```
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    ssh root@${
   
     master_ip} "mkdir -p ~/.kube"
    scp kubectl.kubeconfig root@${
   
     master_ip}:~/.kube/config
    ssh root@${
   
     master_ip} "echo 'export KUBECONFIG=\$HOME/.kube/config' >> ~/.bashrc"
    ssh root@${
   
     master_ip} "echo 'source <(kubectl completion bash)' >> ~/.bashrc"
  done
```

下面命令需要在 `k8s-master01` 和 `k8s-master02` 上配置：

```
[root@k8s-master01 work]# source /usr/share/bash-completion/bash_completion
[root@k8s-master01 work]# source <(kubectl completion bash)
[root@k8s-master01 work]# bash ~/.bashrc
```

# 