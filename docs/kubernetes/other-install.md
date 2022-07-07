---
sidebar_position: 3
---

# 三、安装相关组件

## 1.安装 Kube-APIServer 组件

下载Kubernetes 二进制文件

```
[root@k8s-master01 work]# wget https://storage.googleapis.com/kubernetes-release/release/v1.18.3/kubernetes-server-linux-amd64.tar.gz
[root@k8s-master01 work]# tar -zxf kubernetes-server-linux-amd64.tar.gz
[root@k8s-master01 work]# cd kubernetes
[root@k8s-master01 kubernetes]# tar -zxf kubernetes-src.tar.gz
[root@k8s-master01 kubernetes]# cd ..
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp -rp kubernetes/server/bin/{
   
     apiextensions-apiserver,kube-apiserver,kube-controller-manager,kube-scheduler,kubeadm,kubectl,mounter} root@${
   
     master_ip}:/opt/k8s/bin/
    ssh root@${
   
     master_ip} "chmod +x /opt/k8s/bin/*"
  done
```

### 1）创建 Kubernetes 证书和密钥

```
[root@k8s-master01 work]# cat > kubernetes-csr.json << EOF
{
   
     
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "192.168.1.1",
    "192.168.1.2",
    "${CLUSTER_KUBERNETES_SVC_IP}",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local."
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

### 2）生成证书和密钥

```
[root@k8s-master01 work]# cfssl gencert -ca=/opt/k8s/work/ca.pem -ca-key=/opt/k8s/work/ca-key.pem -config=/opt/k8s/work/ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    ssh root@${
   
     master_ip} "mkdir -p /etc/kubernetes/cert"
    scp kubernetes*.pem root@${
   
     master_ip}:/etc/kubernetes/cert/
  done
```

### 3）配置 Kube-APIServer 审计

创建加密配置文件

```
[root@k8s-master01 work]# cat > encryption-config.yaml << EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: zhangsan
              secret: ${
   
     ENCRYPTION_KEY}
      - identity: {
   
     }
EOF
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp encryption-config.yaml root@${
   
     master_ip}:/etc/kubernetes/encryption-config.yaml
  done
```

创建审计策略文件

```
[root@k8s-master01 work]# cat > audit-policy.yaml << EOF
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
  # The following requests were manually identified as high-volume and low-risk, so drop them.
  - level: None
    resources:
      - group: ""
        resources:
          - endpoints
          - services
          - services/status
    users:
      - 'system:kube-proxy'
    verbs:
      - watch

  - level: None
    resources:
      - group: ""
        resources:
          - nodes
          - nodes/status
    userGroups:
      - 'system:nodes'
    verbs:
      - get

  - level: None
    namespaces:
      - kube-system
    resources:
      - group: ""
        resources:
          - endpoints
    users:
      - 'system:kube-controller-manager'
      - 'system:kube-scheduler'
      - 'system:serviceaccount:kube-system:endpoint-controller'
    verbs:
      - get
      - update

  - level: None
    resources:
      - group: ""
        resources:
          - namespaces
          - namespaces/status
          - namespaces/finalize
    users:
      - 'system:apiserver'
    verbs:
      - get

  # Don't log HPA fetching metrics.
  - level: None
    resources:
      - group: metrics.k8s.io
    users:
      - 'system:kube-controller-manager'
    verbs:
      - get
      - list

  # Don't log these read-only URLs.
  - level: None
    nonResourceURLs:
      - '/healthz*'
      - /version
      - '/swagger*'

  # Don't log events requests.
  - level: None
    resources:
      - group: ""
        resources:
          - events

  # node and pod status calls from nodes are high-volume and can be large, don't log responses for expected updates from nodes
  - level: Request
    omitStages:
      - RequestReceived
    resources:
      - group: ""
        resources:
          - nodes/status
          - pods/status
    users:
      - kubelet
      - 'system:node-problem-detector'
      - 'system:serviceaccount:kube-system:node-problem-detector'
    verbs:
      - update
      - patch

  - level: Request
    omitStages:
      - RequestReceived
    resources:
      - group: ""
        resources:
          - nodes/status
          - pods/status
    userGroups:
      - 'system:nodes'
    verbs:
      - update
      - patch

  # deletecollection calls can be large, don't log responses for expected namespace deletions
  - level: Request
    omitStages:
      - RequestReceived
    users:
      - 'system:serviceaccount:kube-system:namespace-controller'
    verbs:
      - deletecollection

  # Secrets, ConfigMaps, and TokenReviews can contain sensitive & binary data,
  # so only log at the Metadata level.
  - level: Metadata
    omitStages:
      - RequestReceived
    resources:
      - group: ""
        resources:
          - secrets
          - configmaps
      - group: authentication.k8s.io
        resources:
          - tokenreviews
  # Get repsonses can be large; skip them.
  - level: Request
    omitStages:
      - RequestReceived
    resources:
      - group: ""
      - group: admissionregistration.k8s.io
      - group: apiextensions.k8s.io
      - group: apiregistration.k8s.io
      - group: apps
      - group: authentication.k8s.io
      - group: authorization.k8s.io
      - group: autoscaling
      - group: batch
      - group: certificates.k8s.io
      - group: extensions
      - group: metrics.k8s.io
      - group: networking.k8s.io
      - group: policy
      - group: rbac.authorization.k8s.io
      - group: scheduling.k8s.io
      - group: settings.k8s.io
      - group: storage.k8s.io
    verbs:
      - get
      - list
      - watch

  # Default level for known APIs
  - level: RequestResponse
    omitStages:
      - RequestReceived
    resources:
      - group: ""
      - group: admissionregistration.k8s.io
      - group: apiextensions.k8s.io
      - group: apiregistration.k8s.io
      - group: apps
      - group: authentication.k8s.io
      - group: authorization.k8s.io
      - group: autoscaling
      - group: batch
      - group: certificates.k8s.io
      - group: extensions
      - group: metrics.k8s.io
      - group: networking.k8s.io
      - group: policy
      - group: rbac.authorization.k8s.io
      - group: scheduling.k8s.io
      - group: settings.k8s.io
      - group: storage.k8s.io

  # Default level for all other requests.
  - level: Metadata
    omitStages:
      - RequestReceived
EOF
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp audit-policy.yaml root@${
   
     master_ip}:/etc/kubernetes/audit-policy.yaml
  done
```

### 4）配置 Metrics-Server

创建`metrics-server` 的 CA 证书请求文件

```
[root@k8s-master01 work]# cat > proxy-client-csr.json << EOF
{
   
     
  "CN": "system:metrics-server",
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

生成证书和密钥

```
[root@k8s-master01 work]# cfssl gencert -ca=/opt/k8s/work/ca.pem -ca-key=/opt/k8s/work/ca-key.pem -config=/opt/k8s/work/ca-config.json -profile=kubernetes proxy-client-csr.json | cfssljson -bare proxy-client
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp proxy-client*.pem root@${
   
     master_ip}:/etc/kubernetes/cert/
  done
```

### 5）创建启动脚本

```
[root@k8s-master01 work]# cat > kube-apiserver.service.template << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=${
   
     K8S_DIR}/kube-apiserver
ExecStart=/opt/k8s/bin/kube-apiserver \\
  --insecure-port=0 \\
  --secure-port=6443 \\
  --bind-address=##MASTER_IP## \\
  --advertise-address=##MASTER_IP## \\
  --default-not-ready-toleration-seconds=360 \\
  --default-unreachable-toleration-seconds=360 \\
  --feature-gates=DynamicAuditing=true \\
  --max-mutating-requests-inflight=2000 \\
  --max-requests-inflight=4000 \\
  --default-watch-cache-size=200 \\
  --delete-collection-workers=2 \\
  --encryption-provider-config=/etc/kubernetes/encryption-config.yaml \\
  --etcd-cafile=/etc/kubernetes/cert/ca.pem \\
  --etcd-certfile=/etc/kubernetes/cert/kubernetes.pem \\
  --etcd-keyfile=/etc/kubernetes/cert/kubernetes-key.pem \\
  --etcd-servers=${
   
     ETCD_ENDPOINTS} \\
  --tls-cert-file=/etc/kubernetes/cert/kubernetes.pem \\
  --tls-private-key-file=/etc/kubernetes/cert/kubernetes-key.pem \\
  --audit-dynamic-configuration \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-truncate-enabled=true \\
  --audit-log-path=${
   
     K8S_DIR}/kube-apiserver/audit.log \\
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \\
  --profiling \\
  --anonymous-auth=false \\
  --client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --enable-bootstrap-token-auth=true \\
  --requestheader-allowed-names="system:metrics-server" \\
  --requestheader-client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --requestheader-extra-headers-prefix=X-Remote-Extra- \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User \\
  --service-account-key-file=/etc/kubernetes/cert/ca.pem \\
  --authorization-mode=Node,RBAC \\
  --runtime-config=api/all=true \\
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota,NodeRestriction \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --event-ttl=168h \\
  --kubelet-certificate-authority=/etc/kubernetes/cert/ca.pem \\
  --kubelet-client-certificate=/etc/kubernetes/cert/kubernetes.pem \\
  --kubelet-client-key=/etc/kubernetes/cert/kubernetes-key.pem \\
  --kubelet-https=true \\
  --kubelet-timeout=10s \\
  --proxy-client-cert-file=/etc/kubernetes/cert/proxy-client.pem \\
  --proxy-client-key-file=/etc/kubernetes/cert/proxy-client-key.pem \\
  --service-cluster-ip-range=${
   
     SERVICE_CIDR} \\
  --service-node-port-range=${
   
     NODE_PORT_RANGE} \\
  --logtostderr=true \\
  --v=2
Restart=on-failure
RestartSec=10
Type=notify
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

### 6）启动 Kube-APIServer 并验证

```
[root@k8s-master01 work]# for (( A=0; A < 2; A++ ))
  do
    sed -e "s/##MASTER_NAME##/${MASTER_NAMES[A]}/" -e "s/##MASTER_IP##/${MASTER_IPS[A]}/" kube-apiserver.service.template > kube-apiserver-${
   
     MASTER_IPS[A]}.service
  done
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp kube-apiserver-${
   
     master_ip}.service root@${
   
     master_ip}:/etc/systemd/system/kube-apiserver.service
    ssh root@${
   
     master_ip} "mkdir -p ${K8S_DIR}/kube-apiserver"
    ssh root@${
   
     master_ip} "systemctl daemon-reload && systemctl enable kube-apiserver --now"
  done
```

查看Kube-APIServer 写入 ETCD 的数据

```
[root@k8s-master01 work]# ETCDCTL_API=3 etcdctl \
--endpoints=${
   
     ETCD_ENDPOINTS} \
--cacert=/opt/k8s/work/ca.pem \
--cert=/opt/k8s/work/etcd.pem \
--key=/opt/k8s/work/etcd-key.pem \
get /registry/ --prefix --keys-only
```

查看集群信息

```
[root@k8s-master01 work]# kubectl cluster-info
[root@k8s-master01 work]# kubectl get all --all-namespaces
[root@k8s-master01 work]# kubectl get componentstatuses
[root@k8s-master01 work]# netstat -anpt | grep 6443
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)
授予`kube-apiserver` 访问 `kubelet` API 的权限

```
[root@k8s-master01 work]# kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes
```

## 2.安装 Controller Manager 组件

### 1）创建 Controller Manager 证书和密钥

```
[root@k8s-master01 work]# cat > kube-controller-manager-csr.json << EOF
{
   
     
  "CN": "system:kube-controller-manager",
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
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
EOF
```

### 2）生成证书和密钥

```
[root@k8s-master01 work]# cfssl gencert -ca=/opt/k8s/work/ca.pem -ca-key=/opt/k8s/work/ca-key.pem -config=/opt/k8s/work/ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp kube-controller-manager*.pem root@${
   
     master_ip}:/etc/kubernetes/cert/
  done
```

### 3）创建 Kubeconfig 文件

```
[root@k8s-master01 work]# kubectl config set-cluster kubernetes \
--certificate-authority=/opt/k8s/work/ca.pem \
--embed-certs=true \
--server=${
   
     KUBE_APISERVER} \
--kubeconfig=kube-controller-manager.kubeconfig
[root@k8s-master01 work]# kubectl config set-credentials system:kube-controller-manager \
--client-certificate=kube-controller-manager.pem \
--client-key=kube-controller-manager-key.pem \
--embed-certs=true \
--kubeconfig=kube-controller-manager.kubeconfig
[root@k8s-master01 work]# kubectl config set-context system:kube-controller-manager \
--cluster=kubernetes \
--user=system:kube-controller-manager \
--kubeconfig=kube-controller-manager.kubeconfig
[root@k8s-master01 work]# kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp kube-controller-manager.kubeconfig root@${
   
     master_ip}:/etc/kubernetes/
  done
```

### 4）创建启动脚本

```
[root@k8s-master01 work]# cat > kube-controller-manager.service.template << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
WorkingDirectory=${
   
     K8S_DIR}/kube-controller-manager
ExecStart=/opt/k8s/bin/kube-controller-manager \\
  --secure-port=10257 \\
  --bind-address=127.0.0.1 \\
  --profiling \\
  --cluster-name=kubernetes \\
  --controllers=*,bootstrapsigner,tokencleaner \\
  --kube-api-qps=1000 \\
  --kube-api-burst=2000 \\
  --leader-elect \\
  --use-service-account-credentials\\
  --concurrent-service-syncs=2 \\
  --tls-cert-file=/etc/kubernetes/cert/kube-controller-manager.pem \\
  --tls-private-key-file=/etc/kubernetes/cert/kube-controller-manager-key.pem \\
  --authentication-kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \\
  --client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --requestheader-allowed-names="system:metrics-server" \\
  --requestheader-client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --requestheader-extra-headers-prefix="X-Remote-Extra-" \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User \\
  --cluster-signing-cert-file=/etc/kubernetes/cert/ca.pem \\
  --cluster-signing-key-file=/etc/kubernetes/cert/ca-key.pem \\
  --experimental-cluster-signing-duration=87600h \\
  --horizontal-pod-autoscaler-sync-period=10s \\
  --concurrent-deployment-syncs=10 \\
  --concurrent-gc-syncs=30 \\
  --node-cidr-mask-size=24 \\
  --service-cluster-ip-range=${
   
     SERVICE_CIDR} \\
  --cluster-cidr=${
   
     CLUSTER_CIDR} \\
  --pod-eviction-timeout=6m \\
  --terminated-pod-gc-threshold=10000 \\
  --root-ca-file=/etc/kubernetes/cert/ca.pem \\
  --service-account-private-key-file=/etc/kubernetes/cert/ca-key.pem \\
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \\
  --logtostderr=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 4）启动并验证

```
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp kube-controller-manager.service.template root@${
   
     master_ip}:/etc/systemd/system/kube-controller-manager.service
    ssh root@${
   
     master_ip} "mkdir -p ${K8S_DIR}/kube-controller-manager"
    ssh root@${
   
     master_ip} "systemctl daemon-reload && systemctl enable kube-controller-manager --now"
  done
```

查看输出的 Metrics

```
[root@k8s-master01 work]# curl -s --cacert /opt/k8s/work/ca.pem --cert /opt/k8s/work/admin.pem --key /opt/k8s/work/admin-key.pem https://127.0.0.1:10257/metrics | head
```

查看权限

```
[root@k8s-master01 work]# kubectl describe clusterrole system:kube-controller-manager
[root@k8s-master01 work]# kubectl get clusterrole | grep controller
[root@k8s-master01 work]# kubectl describe clusterrole system:controller:deployment-controller
```

查看当前的 Leader

```
[root@k8s-master01 work]# kubectl get endpoints kube-controller-manager --namespace=kube-system -o yaml
```

## 3.安装 Kube-Scheduler 组件

### 1）创建 Kube-Scheduler 证书和密钥

```
[root@k8s-master01 work]# cat > kube-scheduler-csr.json << EOF
{
   
     
  "CN": "system:kube-scheduler",
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
      "O": "system:kube-scheduler",
      "OU": "System"
    }
  ]
}
EOF
```

### 2）生成证书和密钥

```
[root@k8s-master01 work]# cfssl gencert -ca=/opt/k8s/work/ca.pem -ca-key=/opt/k8s/work/ca-key.pem -config=/opt/k8s/work/ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp kube-scheduler*.pem root@${
   
     master_ip}:/etc/kubernetes/cert/
  done
```

### 3）创建 Kubeconfig 文件

```
[root@k8s-master01 work]# kubectl config set-cluster kubernetes \
--certificate-authority=/opt/k8s/work/ca.pem \
--embed-certs=true \
--server=${
   
     KUBE_APISERVER} \
--kubeconfig=kube-scheduler.kubeconfig
[root@k8s-master01 work]# kubectl config set-credentials system:kube-scheduler \
--client-certificate=kube-scheduler.pem \
--client-key=kube-scheduler-key.pem \
--embed-certs=true \
--kubeconfig=kube-scheduler.kubeconfig
[root@k8s-master01 work]# kubectl config set-context system:kube-scheduler \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=kube-scheduler.kubeconfig
[root@k8s-master01 work]# kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp kube-scheduler.kubeconfig root@${
   
     master_ip}:/etc/kubernetes/
  done
```

### 4）创建 Kube-Scheduler 配置文件

```
[root@k8s-master01 work]# cat > kube-scheduler.yaml.template << EOF
apiVersion: kubescheduler.config.k8s.io/v1alpha1
kind: KubeSchedulerConfiguration
bindTimeoutSeconds: 600
clientConnection:
  burst: 200
  kubeconfig: "/etc/kubernetes/kube-scheduler.kubeconfig"
  qps: 100
enableContentionProfiling: false
enableProfiling: true
hardPodAffinitySymmetricWeight: 1
healthzBindAddress: 127.0.0.1:10251
leaderElection:
  leaderElect: true
metricsBindAddress: 127.0.0.1:10251
EOF
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp kube-scheduler.yaml.template root@${
   
     master_ip}:/etc/kubernetes/kube-scheduler.yaml
  done
```

### 5）创建启动脚本

```
[root@k8s-master01 work]# cat > kube-scheduler.service.template << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
WorkingDirectory=${
   
     K8S_DIR}/kube-scheduler
ExecStart=/opt/k8s/bin/kube-scheduler \\
  --port=0 \\
  --secure-port=10259 \\
  --bind-address=127.0.0.1 \\
  --config=/etc/kubernetes/kube-scheduler.yaml \\
  --tls-cert-file=/etc/kubernetes/cert/kube-scheduler.pem \\
  --tls-private-key-file=/etc/kubernetes/cert/kube-scheduler-key.pem \\
  --authentication-kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \\
  --client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --requestheader-allowed-names="system:metrics-server" \\
  --requestheader-client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --requestheader-extra-headers-prefix="X-Remote-Extra-" \\
  --requestheader-group-headers=X-Remote-Group \\
  --requestheader-username-headers=X-Remote-User \\
  --authorization-kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \\
  --logtostderr=true \\
  --v=2
Restart=always
RestartSec=5
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
EOF
```

### 6）启动并验证

```
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    scp kube-scheduler.service.template root@${
   
     master_ip}:/etc/systemd/system/kube-scheduler.service
    ssh root@${
   
     master_ip} "mkdir -p ${K8S_DIR}/kube-scheduler"
    ssh root@${
   
     master_ip} "systemctl daemon-reload && systemctl enable kube-scheduler --now"
  done
[root@k8s-master01 work]# netstat -nlpt | grep kube-schedule
```

- 10251：接收 http 请求，非安全端口，不需要认证授权；
- 10259：接收 https 请求，安全端口，需要认认证授权（两个接口都对外提供 /metrics 和 /healthz 的访问）

查看输出的 Metrics

```
[root@k8s-master01 work]# curl -s --cacert /opt/k8s/work/ca.pem --cert /opt/k8s/work/admin.pem --key /opt/k8s/work/admin-key.pem https://127.0.0.1:10257/metrics | head
```

查看权限

```
[root@k8s-master01 work]# kubectl describe clusterrole system:kube-controller-manager
[root@k8s-master01 work]# kubectl get clusterrole | grep controller
[root@k8s-master01 work]# kubectl describe clusterrole system:controller:deployment-controller
```

查看当前的 Leader

```
[root@k8s-master01 work]# kubectl get endpoints kube-controller-manager --namespace=kube-system -o yaml
```

## 4.安装 Kubelet 组件

```
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    scp kubernetes/server/bin/kubelet root@${
   
     all_ip}:/opt/k8s/bin/
    ssh root@${
   
     all_ip} "chmod +x /opt/k8s/bin/*"
  done
[root@k8s-master01 work]# for all_name in ${
   
     ALL_NAMES[@]}
  do
    echo ">>> ${all_name}"
    export BOOTSTRAP_TOKEN=$(kubeadm token create \
      --description kubelet-bootstrap-token \
      --groups system:bootstrappers:${
   
     all_name} \
      --kubeconfig ~/.kube/config)
    kubectl config set-cluster kubernetes \
      --certificate-authority=/etc/kubernetes/cert/ca.pem \
      --embed-certs=true \
      --server=${
   
     KUBE_APISERVER} \
      --kubeconfig=kubelet-bootstrap-${
   
     all_name}.kubeconfig
    kubectl config set-credentials kubelet-bootstrap \
      --token=${
   
     BOOTSTRAP_TOKEN} \
      --kubeconfig=kubelet-bootstrap-${
   
     all_name}.kubeconfig
    kubectl config set-context default \
      --cluster=kubernetes \
      --user=kubelet-bootstrap \
      --kubeconfig=kubelet-bootstrap-${
   
     all_name}.kubeconfig
    kubectl config use-context default --kubeconfig=kubelet-bootstrap-${
   
     all_name}.kubeconfig
  done
[root@k8s-master01 work]# kubeadm token list --kubeconfig ~/.kube/config     # 查看 Kubeadm 为各节点创建的 Token
[root@k8s-master01 work]# kubectl get secrets -n kube-system | grep bootstrap-token   # 查看各 Token 关联的 Secret
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==) 

```
[root@k8s-master01 work]# for all_name in ${
   
     ALL_NAMES[@]}
  do
    echo ">>> ${all_name}"
    scp kubelet-bootstrap-${
   
     all_name}.kubeconfig root@${
   
     all_name}:/etc/kubernetes/kubelet-bootstrap.kubeconfig
  done
```

创建Kubelet 参数配置文件

```
[root@k8s-master01 work]# cat > kubelet-config.yaml.template << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: "##ALL_IP##"
staticPodPath: ""
syncFrequency: 1m
fileCheckFrequency: 20s
httpCheckFrequency: 20s
staticPodURL: ""
port: 10250
readOnlyPort: 0
rotateCertificates: true
serverTLSBootstrap: true
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/cert/ca.pem"
authorization:
  mode: Webhook
registryPullQPS: 0
registryBurst: 20
eventRecordQPS: 0
eventBurst: 20
enableDebuggingHandlers: true
enableContentionProfiling: true
healthzPort: 10248
healthzBindAddress: "##ALL_IP##"
clusterDomain: "${CLUSTER_DNS_DOMAIN}"
clusterDNS:
  - "${CLUSTER_DNS_SVC_IP}"
nodeStatusUpdateFrequency: 10s
nodeStatusReportFrequency: 1m
imageMinimumGCAge: 2m
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
volumeStatsAggPeriod: 1m
kubeletCgroups: ""
systemCgroups: ""
cgroupRoot: ""
cgroupsPerQOS: true
cgroupDriver: cgroupfs
runtimeRequestTimeout: 10m
hairpinMode: promiscuous-bridge
maxPods: 220
podCIDR: "${CLUSTER_CIDR}"
podPidsLimit: -1
resolvConf: /etc/resolv.conf
maxOpenFiles: 1000000
kubeAPIQPS: 1000
kubeAPIBurst: 2000
serializeImagePulls: false
evictionHard:
  memory.available:  "100Mi"
nodefs.available:  "10%"
nodefs.inodesFree: "5%"
imagefs.available: "15%"
evictionSoft: {
   
     }
enableControllerAttachDetach: true
failSwapOn: true
containerLogMaxSize: 20Mi
containerLogMaxFiles: 10
systemReserved: {
   
     }
kubeReserved: {
   
     }
systemReservedCgroup: ""
kubeReservedCgroup: ""
enforceNodeAllocatable: ["pods"]
EOF
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    sed -e "s/##ALL_IP##/${all_ip}/" kubelet-config.yaml.template > kubelet-config-${
   
     all_ip}.yaml.template
    scp kubelet-config-${
   
     all_ip}.yaml.template root@${
   
     all_ip}:/etc/kubernetes/kubelet-config.yaml
  done
```

### 1）创建 Kubelet 启动脚本

```
[root@k8s-master01 work]# cat > kubelet.service.template << EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=${
   
     K8S_DIR}/kubelet
ExecStart=/opt/k8s/bin/kubelet \\
  --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \\
  --cert-dir=/etc/kubernetes/cert \\
  --cgroup-driver=cgroupfs \\
  --cni-conf-dir=/etc/cni/net.d \\
  --container-runtime=docker \\
  --container-runtime-endpoint=unix:///var/run/dockershim.sock \\
  --root-dir=${
   
     K8S_DIR}/kubelet \\
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
  --config=/etc/kubernetes/kubelet-config.yaml \\
  --hostname-override=##ALL_NAME## \\
  --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause-amd64:3.2 \\
  --image-pull-progress-deadline=15m \\
  --volume-plugin-dir=${
   
     K8S_DIR}/kubelet/kubelet-plugins/volume/exec/ \\
  --logtostderr=true \\
  --v=2
Restart=always
RestartSec=5
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
EOF
[root@k8s-master01 work]# for all_name in ${
   
     ALL_NAMES[@]}
  do
    echo ">>> ${all_name}"
    sed -e "s/##ALL_NAME##/${all_name}/" kubelet.service.template > kubelet-${
   
     all_name}.service
    scp kubelet-${
   
     all_name}.service root@${
   
     all_name}:/etc/systemd/system/kubelet.service
  done
```

### 2）启动并验证

授权

```
[root@k8s-master01 ~]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --group=system:bootstrappers
```

启动Kubelet

```
[root@k8s-master01 work]# for all_name in ${
   
     ALL_NAMES[@]}
  do
    echo ">>> ${all_name}"
    ssh root@${
   
     all_name} "mkdir -p ${K8S_DIR}/kubelet/kubelet-plugins/volume/exec/"
    ssh root@${
   
     all_name} "systemctl daemon-reload && systemctl enable kubelet --now"
  done
```

查看Kubelet 服务

```
[root@k8s-master01 work]# for all_name in ${
   
     ALL_NAMES[@]}
  do
    echo ">>> ${all_name}"
    ssh root@${
   
     all_name} "systemctl status kubelet | grep active"
  done
[root@k8s-master01 work]# kubectl get csr         # 因为我们还没做认证. 所以显示 Pengding 状态
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==) 

### 3）Approve CSR 请求

自动Approve CSR 请求（创建三个 ClusterRoleBinding，分别用于自动 `approve client` `renew client` `renew server`证书）

```
[root@k8s-master01 work]# cat > csr-crb.yaml << EOF
 # Approve all CSRs for the group "system:bootstrappers"
 kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: auto-approve-csrs-for-group
 subjects:
 - kind: Group
   name: system:bootstrappers
   apiGroup: rbac.authorization.k8s.io
 roleRef:
   kind: ClusterRole
   name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
   apiGroup: rbac.authorization.k8s.io
---
 # To let a node of the group "system:nodes" renew its own credentials
 kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: node-client-cert-renewal
 subjects:
 - kind: Group
   name: system:nodes
   apiGroup: rbac.authorization.k8s.io
 roleRef:
   kind: ClusterRole
   name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
   apiGroup: rbac.authorization.k8s.io
---
# A ClusterRole which instructs the CSR approver to approve a node requesting a
# serving cert matching its client cert.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: approve-node-server-renewal-csr
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/selfnodeserver"]
  verbs: ["create"]
---
 # To let a node of the group "system:nodes" renew its own server credentials
 kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: node-server-cert-renewal
 subjects:
 - kind: Group
   name: system:nodes
   apiGroup: rbac.authorization.k8s.io
 roleRef:
   kind: ClusterRole
   name: approve-node-server-renewal-csr
   apiGroup: rbac.authorization.k8s.io
EOF
[root@k8s-master01 work]# kubectl apply -f csr-crb.yaml
```

验证（等待一段时间 `1 ~ 5` 分钟)，三个节点的 CSR 都自动 `approved`）

```
[root@k8s-master01 work]# kubectl get csr | grep boot    # 等待一段时间 (1-10 分钟)，三个节点的 CSR 都自动 approved
[root@k8s-master01 work]# kubectl get nodes       # 所有节点均 Ready
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==) 

```
[root@k8s-master01 ~]# ls -l /etc/kubernetes/kubelet.kubeconfig
[root@k8s-master01 ~]# ls -l /etc/kubernetes/cert/ | grep kubelet
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==) 

### 4）手动 Approve Server Cert Csr

基于安全性考虑，CSR `approving controllers` 不会自动 `approve kubelet server` 证书签名请求，需要手动 `approve`

```
[root@k8s-master01 ~]# kubectl get csr | grep node
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==) 

```
[root@k8s-master01 ~]# kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve
[root@k8s-master01 ~]# ls -l /etc/kubernetes/cert/kubelet-*
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==) 

### 5）Kubelet API 接口配置

Kubelet API 认证和授权

```
[root@k8s-master01 ~]# curl -s --cacert /etc/kubernetes/cert/ca.pem https://192.168.1.1:10250/metrics
[root@k8s-master01 ~]# curl -s --cacert /etc/kubernetes/cert/ca.pem -H "Authorization: Bearer 123456" https://192.168.1.1:10250/metrics
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)
证书认证和授权

```
// 默认权限不足
[root@k8s-master01 ~]# curl -s --cacert /etc/kubernetes/cert/ca.pem --cert /etc/kubernetes/cert/kube-controller-manager.pem --key /etc/kubernetes/cert/kube-controller-manager-key.pem https://192.168.1.1:10250/metrics
// 使用最高权限的 admin
[root@k8s-master01 ~]# curl -s --cacert /etc/kubernetes/cert/ca.pem --cert /opt/k8s/work/admin.pem --key /opt/k8s/work/admin-key.pem https://192.168.1.1:10250/metrics | head
```

创建Bear Token 认证和授权

```
[root@k8s-master01 ~]# kubectl create serviceaccount kubelet-api-test
[root@k8s-master01 ~]# kubectl create clusterrolebinding kubelet-api-test --clusterrole=system:kubelet-api-admin --serviceaccount=default:kubelet-api-test
[root@k8s-master01 ~]# SECRET=$(kubectl get secrets | grep kubelet-api-test | awk '{print $1}')
[root@k8s-master01 ~]# TOKEN=$(kubectl describe secret ${
   
     SECRET} | grep -E '^token' | awk '{print $2}')
[root@k8s-master01 ~]# echo ${
   
     TOKEN}
[root@k8s-master01 ~]# curl -s --cacert /etc/kubernetes/cert/ca.pem -H "Authorization: Bearer ${TOKEN}" https://192.168.1.1:10250/metrics | head
```

## 5.安装 Kube-Proxy 组件

Kube-Proxy 运行在所有主机上，用来监听 APIServer 中的 Service 和 Endpoint 的变化情况，并创建路由规则来提供服务 IP 和负载均衡功能。

```
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    scp kubernetes/server/bin/kube-proxy root@${
   
     all_ip}:/opt/k8s/bin/
    ssh root@${
   
     all_ip} "chmod +x /opt/k8s/bin/*"
  done
```

### 1）创建 Kube-Proxy 证书和密钥

创建Kube-Proxy 的 CA 证书请求文件

```
[root@k8s-master01 work]# cat > kube-proxy-csr.json << EOF
{
   
     
  "CN": "system:kube-proxy",
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
[root@k8s-master01 work]# cfssl gencert -ca=/opt/k8s/work/ca.pem -ca-key=/opt/k8s/work/ca-key.pem -config=/opt/k8s/work/ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
```

### 3）创建 Kubeconfig 文件

```
[root@k8s-master01 work]# kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/k8s/work/ca.pem \
  --embed-certs=true \
  --server=${
   
     KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig

[root@k8s-master01 work]# kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

[root@k8s-master01 work]# kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

[root@k8s-master01 work]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    scp kube-proxy.kubeconfig root@${
   
     all_ip}:/etc/kubernetes/
  done
```

### 4）创建 Kube-Proxy 配置文件

```
[root@k8s-master01 work]# cat > kube-proxy-config.yaml.template << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  burst: 200
  kubeconfig: "/etc/kubernetes/kube-proxy.kubeconfig"
  qps: 100
bindAddress: ##ALL_IP##
healthzBindAddress: ##ALL_IP##:10256
metricsBindAddress: ##ALL_IP##:10249
enableProfiling: true
clusterCIDR: ${
   
     CLUSTER_CIDR}
hostnameOverride: ##ALL_NAME##
mode: "ipvs"
portRange: ""
kubeProxyIPTablesConfiguration:
  masqueradeAll: false
kubeProxyIPVSConfiguration:
  scheduler: rr
  excludeCIDRs: []
EOF
[root@k8s-master01 work]# for (( i=0; i < 3; i++ ))
  do
    echo ">>> ${ALL_NAMES[i]}"
    sed -e "s/##ALL_NAME##/${ALL_NAMES[i]}/" -e "s/##ALL_IP##/${ALL_IPS[i]}/" kube-proxy-config.yaml.template > kube-proxy-config-${
   
     ALL_NAMES[i]}.yaml.template
    scp kube-proxy-config-${
   
     ALL_NAMES[i]}.yaml.template root@${
   
     ALL_NAMES[i]}:/etc/kubernetes/kube-proxy-config.yaml
  done
```

### 4）创建启动脚本

```
[root@k8s-master01 work]# cat > kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=${
   
     K8S_DIR}/kube-proxy
ExecStart=/opt/k8s/bin/kube-proxy \\
  --config=/etc/kubernetes/kube-proxy-config.yaml \\
  --logtostderr=true \\
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
[root@k8s-master01 work]# for all_name in ${
   
     ALL_NAMES[@]}
  do
    echo ">>> ${all_name}"
    scp kube-proxy.service root@${
   
     all_name}:/etc/systemd/system/
  done
```

### 5）启动并验证

```
[root@k8s-master01 work]# for all_ip in ${
   
     ALL_IPS[@]}
  do
    echo ">>> ${all_ip}"
    ssh root@${
   
     all_ip} "mkdir -p ${K8S_DIR}/kube-proxy"
    ssh root@${
   
     all_ip} "modprobe ip_vs_rr"
    ssh root@${
   
     all_ip} "systemctl daemon-reload && systemctl enable kube-proxy --now"
  done
```

查看`ipvs` 路由规则

```
[root@k8s-master01 work]# ipvsadm -ln
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/GpcH5Yqqj0n9Cliag8kbwng634qNfmqFjG7AgZtVk1h0M13PY5PCBToFibso96WL9KzUZcbfnUnRoPw4SXN1xeYw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
**问题：** 当我们在启动 `kube-proxy` 组件后，通过 `systemctl` 查看该组件状态时，出现如下错误

```
Not using --random-fully in the MASQUERADE rule for iptables because the local version of iptables does not support it
```

上面报错是因为我们的 `iptables` 版本不支持 `--random-fully` 配置（1.6.2 版本上支持），所以我们需要对 `iptables` 进行升级操作。

```
[root@master01 work]# wget https://www.netfilter.org/projects/iptables/files/iptables-1.6.2.tar.bz2 --no-check-certificate
[root@master01 work]# for all_name in ${
   
     ALL_NAMES[@]}
  do
    echo ">>> ${all_name}"
    scp iptables-1.6.2.tar.bz2 root@${
   
     all_name}:/root/
    ssh root@${
   
     all_name} "yum -y install gcc make libnftnl-devel libmnl-devel autoconf automake libtool bison flex libnetfilter_conntrack-devel libnetfilter_queue-devel libpcap-devel bzip2"
    ssh root@${
   
     all_name} "export LC_ALL=C && tar -xf iptables-1.6.2.tar.bz2 && cd iptables-1.6.2 && ./autogen.sh && ./configure && make && make install"
    ssh root@${
   
     all_name} "systemctl daemon-reload && systemctl restart kubelet && systemctl restart kube-proxy"
  done
```

## 6.安装 CoreDNS 插件

### 1）修改 Coredns 配置

```
[root@k8s-master01 ~]# cd /opt/k8s/work/kubernetes/cluster/addons/dns/coredns
[root@k8s-master01 coredns]# cp coredns.yaml.base coredns.yaml
[root@k8s-master01 coredns]# sed -i -e "s/__PILLAR__DNS__DOMAIN__/${CLUSTER_DNS_DOMAIN}/" -e "s/__PILLAR__DNS__SERVER__/${CLUSTER_DNS_SVC_IP}/" -e "s/__PILLAR__DNS__MEMORY__LIMIT__/200Mi/" coredns.yaml
```

### 2）创建 Coredns 并启动

配置调度策略

```
[root@k8s-master01 coredns]# kubectl label nodes k8s-master01 node-role.kubernetes.io/master=true
[root@k8s-master01 coredns]# kubectl label nodes k8s-master02 node-role.kubernetes.io/master=true
[root@k8s-master01 coredns]# vim coredns.yaml
......
apiVersion: apps/v1
kind: Deployment
......
spec:
  replicas: 2               # 配置成两个副本
......
      tolerations:
        - key: "node-role.kubernetes.io/master"
          operator: "Equal"
          value: ""
          effect: NoSchedule
      nodeSelector:
        node-role.kubernetes.io/master: "true"
......
[root@k8s-master01 coredns]# kubectl create -f coredns.yaml
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==) 

```
kubectl describe pod Pod-Name -n kube-system           # Pod-Name 你们需要换成自己的
```

因为上面镜像使用的是 K8s 官方的镜像（国外），所以可能会出现：

```
Normal   BackOff    72s (x6 over 3m47s)   kubelet, k8s-master01  Back-off pulling image "k8s.gcr.io/coredns:1.6.5"
Warning  Failed     57s (x7 over 3m47s)   kubelet, k8s-master01  Error: ImagePullBackOff
```

- 出现如上问题后，我们可以通过拉取其它仓库中的镜像，拉取完后重新打个标签即可。

```
如：docker pull k8s.gcr.io/coredns:1.6.5

我们可以：
docker pull registry.aliyuncs.com/google_containers/coredns:1.6.5
docker tag registry.aliyuncs.com/google_containers/coredns:1.6.5 k8s.gcr.io/coredns:1.6.5
```

### 3）验证

```
[root@k8s-master01 coredns]# kubectl run -it --rm test-dns --image=busybox:1.28.4 sh
If you don't see a command prompt, try pressing enter.
/ # 
/ # nslookup kubernetes
Server:    10.20.0.254
Address 1: 10.20.0.254 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.20.0.1 kubernetes.default.svc.cluster.local
```

## 7.安装 Dashboard 仪表盘

```
[root@k8s-master01 coredns]# cd /opt/k8s/work/
[root@k8s-master01 work]# mkdir metrics
[root@k8s-master01 work]# cd metrics/
[root@k8s-master01 metrics]# wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
[root@k8s-master01 metrics]# vim components.yaml
......
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  replicas: 2            # 修改副本数
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      hostNetwork: true          # 配置主机网络
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {
   
     }
      containers:
      - name: metrics-server
        image: registry.aliyuncs.com/google_containers/metrics-server-amd64:v0.3.6  # 修改镜像名
        imagePullPolicy: IfNotPresent
        args:
          - --cert-dir=/tmp
          - --secure-port=4443
          - --kubelet-insecure-tls       # 新加的
          - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP # 新加的
......
[root@k8s-master01 metrics]# kubectl create -f components.yaml 
```

验证：![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### 1）创建证书

```
[root@k8s-master01 metrics]# cd /opt/k8s/work/
[root@k8s-master01 work]# mkdir -p /opt/k8s/work/dashboard/certs
[root@k8s-master01 work]# openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=ZheJiang/L=HangZhou/O=Xianghy/OU=Xianghy/CN=k8s.odocker.com"
[root@k8s-master01 work]# for master_ip in ${
   
     MASTER_IPS[@]}
  do
    echo ">>> ${master_ip}"
    ssh root@${
   
     master_ip} "mkdir -p /opt/k8s/work/dashboard/certs" 
    scp tls.* root@${
   
     master_ip}:/opt/k8s/work/dashboard/certs/
  done
```

### 2）修改 Dashboard 配置

手动创建 Secret

```
[root@master01 ~]# kubectl create namespace kubernetes-dashboard
[root@master01 ~]# kubectl create secret generic kubernetes-dashboard-certs --from-file=/opt/k8s/work/dashboard/certs -n kubernetes-dashboard
```

修改Dashboard 配置（你们可以通过这个地址来看 Dashboard 的 `yaml` 文件：传送门）

```
[root@k8s-master01 work]# cd dashboard/
[root@k8s-master01 dashboard]# vim dashboard.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30080
  selector:
    k8s-app: kubernetes-dashboard
---
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""
---
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque
---
kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard
---
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.0.0-beta8
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            - --tls-key-file=tls.key
            - --tls-cert-file=tls.crt
            - --token-ttl=3600
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {
   
     }
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "beta.kubernetes.io/os": linux
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper
---
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      containers:
        - name: dashboard-metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "beta.kubernetes.io/os": linux
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {
   
     }
[root@k8s-master01 dashboard]# kubectl create -f dashboard.yaml          
```

创建管理员账户

```
[root@k8s-master01 dashboard]# kubectl create serviceaccount admin-user -n kubernetes-dashboard
[root@k8s-master01 dashboard]# kubectl create clusterrolebinding admin-user --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:admin-user
```

### 3）验证

获取登录令牌

```
[root@k8s-master01 dashboard]# kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/GpcH5Yqqj0n9Cliag8kbwng634qNfmqFjd7xna7LNnkXjhCtUickcfnbOgOnAibRiaica1mGlFobNptTAbhEiaWnZNpw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
访问：`https://192.168.1.1:30080`
![图片](https://mmbiz.qpic.cn/mmbiz_png/GpcH5Yqqj0n9Cliag8kbwng634qNfmqFjRgv8aG1HI87QCkKoJicn75IXPFrVl8zxvaZuVu2L9YibNeey7AZJUVicA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
![图片](https://mmbiz.qpic.cn/mmbiz_png/GpcH5Yqqj0n9Cliag8kbwng634qNfmqFj7kwBMxjjPuomuTbIe30CS9JhWTsMzsjyKiacQiacAIW57xpjaMmWek7g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)
到此，我们的 Kubernetes 便搭建完成了，如果你们在安装中出现问题，可以通过下面的推广信息来联系博主共同探讨。