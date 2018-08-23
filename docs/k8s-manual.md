# 手动部署Kubernetes:
- 部署Kubernetes-v1.10.3版本。

## 版本明细: Release-v1.10.3
- OS:           CentOS Linux release 7.4.1708 (Core)
- kernel:       3.10.0-693.el7.x86_64
- Kubernetes：  v1.10.3
- Etcd:         v3.3.1
- Docker:       18.03.1-ce
- Flannel:      v0.10.0
- CNI-Plugins:  v0.7.0
- cfssl:        v1.2

## 部署准备: 至少三个节点，请配置好主机名解析(必备)

# 使用手册
<table border="">
    <tr>
        <td><strong>手动部署</strong></td>
        <td><a>1.系统初始化</a></td>
        <td><a>2.CA证书制作</a></td>
        <td><a>3.ETCD集群部署</a></td>
        <td><a>4.Master节点部署</a></td>
        <td><a>5.Node节点部署</a></td>
        <td><a>6.Flannel部署</a></td>
        <td><a>7.创建k8s应用</a></td>
		<td><a>8.必备插件部署</a></td>
    </tr>
    <tr>
        <td><strong>必备插件</strong></td>
        <td><a>1.CoreDNS部署</a></td>
        <td><a>2.Heapster部署</a></td>
        <td><a>3.Ingress部署</a></td>
        <td><a>4.Dashboard部署</a></td>
    </tr>
</table>

## k8s服务列表
### master服务列表:
```
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
systemctl status flannel
systemctl status docker

systemctl restart kube-apiserver
systemctl restart kube-controller-manager
systemctl restart kube-scheduler
systemctl restart flannel
systemctl restart docker
```

### node服务列表:
```
systemctl status kubelet
systemctl status kube-proxy
systemctl status flannel
systemctl status docker

systemctl restart kubelet
systemctl restart kube-proxy
systemctl restart flannel
systemctl restart docker
```

## 1. 系统初始化 (这里不在演示操作):
1. 规范主机名;
2. 设置/etc/hosts主机名内部解析;
3. 关闭SELinux和防火墙;
4. ssh密钥连接;
5. 设置时间同步;

### 1.1 机器分配情况:
```
/etc/hosts:
10.11.40.69    k8s-master     k8s-master.myunsheji.com
10.11.40.71    k8s-node1      k8s-node1.myunsheji.com
10.11.40.72    k8s-node2      k8s-node2.myunsheji.com
10.11.40.73    k8s-node3      k8s-node3.myunsheji.com
```

### 1.2 安装docker (所有节点操作):
#### 1）配置docker国内源:
```
curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

#### 2）安装docker:
```
yum install -y docker-ce-18.03.1.ce-1.el7.centos.x86_64
```

#### 3）设置私有仓库和加速器:
```
mkdir -p /etc/docker
cat <<'EOF'>/etc/docker/daemon.json
{
  "registry-mirrors": ["https://7caofi77.mirror.aliyuncs.com"],
  "insecure-registries": ["10.11.40.59:8082"]
}
EOF

在docker服务启动文件[Service]字段下增加一行:
vim /usr/lib/systemd/system/docker.service:
EnvironmentFile=-/etc/docker/daemon.json
```

#### 4）启动docker服务:
```
systemctl daemon-reload
systemctl enable docker
systemctl start docker
systemctl status docker
```

#### 5) 登录docker私有仓库方式:
```
echo "rqKwWsB+Y1HlAdByT3bT"|docker login -u docker-dev --password-stdin 10.11.40.59:8082
```

### 1.3 创建kubernetes存放位置目录结构:
```
mkdir -p /opt/kubernetes/{cfg,bin,ssl,log}
```

### 1.4 从FTP上下载部署所需软件内容并解压 (master节点即可):
```
wget -c ftp://shejiyun@ftp.myunsheji.com/dropbox/li.zhiqiang/k8s/k8s-v1.10.3-manual.tar.gz --ftp-user=shejiyun --ftp-password=P@ssw0rd!
```

## 2. CA证书制作 (master节点操作)
#### 2.1 安装cfssl生成证书工具:
```
cd files/cfssl-1.2
chmod +x cfssl*
cp cfssl-certinfo_linux-amd64 /opt/kubernetes/bin/cfssl-certinfo
cp cfssljson_linux-amd64  /opt/kubernetes/bin/cfssljson
cp cfssl_linux-amd64  /opt/kubernetes/bin/cfssl

cat <<'EOF'>>/etc/profile
##kubernetes
export kubernetes_home="/opt/kubernetes"
export PATH="$kubernetes_home/bin:$PATH"
EOF

source /etc/profile

PS: 将以上内容拷贝到其余节点相同位置上;
```

#### 2.2 初始化cfssl:
```
mkdir ~/ssl
cd ~/ssl
cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json
```

##### 1) 创建用来生成CA文件的JSON配置文件:
```
cat <<'EOF'>~/ssl/ca-config.json
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
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

##### 2) 创建用来生成CA证书签名请求CSR的JSON配置文件:
```
cat <<'EOF'>~/ssl/ca-csr.json
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
      "OU": "System"
    }
  ]
}
EOF
```

##### 3) 生成CA证书和密钥:
```
cd ~/ssl/
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

##### 4) 将证书分发到所有节点上:
```
scp ca.csr ca.pem ca-key.pem ca-config.json /opt/kubernetes/ssl
```

## 3. ETCD集群部署
#### 3.1 将命令分发到所有节点上:
```
cd files/etcd-v3.3.1-linux-amd64
scp etcd* /opt/kubernetes/bin/
```

#### 3.2 创建etcd证书签名请求:
```
cat <<'EOF'>~/ssl/etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
"10.11.40.71",
"10.11.40.72",
"10.11.40.73"
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
      "OU": "System"
    }
  ]
}
EOF
```

#### 3.3 生成etcd证书和私钥:
```
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
  -ca-key=/opt/kubernetes/ssl/ca-key.pem \
  -config=/opt/kubernetes/ssl/ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```

#### 3.4 将生成内容分发到所有node节点:
```
scp cp etcd*.pem /opt/kubernetes/ssl
```

#### 3.5 设置ETCD配置文件 (node节点操作):
```
例:
k8s-node1:

cat <<'EOF'>/opt/kubernetes/cfg/etcd.conf
#[member]
ETCD_NAME="etcd-node1"		\\名称需要修改;
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
#ETCD_SNAPSHOT_COUNTER="10000"
#ETCD_HEARTBEAT_INTERVAL="100"
#ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="https://10.11.40.71:2380"				\\地址需要修改;
ETCD_LISTEN_CLIENT_URLS="https://10.11.40.71:2379,https://127.0.0.1:2379"	\\地址需要修改;
#ETCD_MAX_SNAPSHOTS="5"
#ETCD_MAX_WALS="5"
#ETCD_CORS=""
#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.11.40.71:2380"	\\地址需要修改;
# if you use different ETCD_NAME (e.g. test),
# set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="etcd-node1=https://10.11.40.71:2380,etcd-node2=https://10.11.40.72:2380,etcd-node3=https://10.11.40.73:2380"	\\地址需要修改;
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://10.11.40.71:2379"		\\地址需要修改;
#[security]
CLIENT_CERT_AUTH="true"
ETCD_CA_FILE="/opt/kubernetes/ssl/ca.pem"
ETCD_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
ETCD_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"
PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_CA_FILE="/opt/kubernetes/ssl/ca.pem"
ETCD_PEER_CERT_FILE="/opt/kubernetes/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/opt/kubernetes/ssl/etcd-key.pem"
EOF

PS: 其余节点自行操作。
```

#### 3.6 创建ETCD系统服务 (node节点操作):
```
cat <<'EOF'>/etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/var/lib/etcd
EnvironmentFile=-/opt/kubernetes/cfg/etcd.conf
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /opt/kubernetes/bin/etcd"
Type=notify

[Install]
WantedBy=multi-user.target
EOF
```

#### 3.7 重新加载系统服务 (node节点操作):
```
创建etcd存储目录并启动etcd:
mkdir -p /var/lib/etcd

systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```

#### 3.8 验证集群状态:
```
etcdctl --endpoints=https://10.11.40.71:2379 \
  --ca-file=/opt/kubernetes/ssl/ca.pem \
  --cert-file=/opt/kubernetes/ssl/etcd.pem \
  --key-file=/opt/kubernetes/ssl/etcd-key.pem cluster-health
```

## 4. Master节点部署 (master节点操作)
### 4.1 部署KubernetesAPI服务:
#### 4.1.1 部署KubernetesAPI服务:
```
cd files/k8s-v1.10.3/bin
cp server/bin/kube-apiserver kube-controller-manager kube-scheduler /opt/kubernetes/bin/
```

#### 4.1.2 创建生成CSR的JSON配置文件:
```
cat <<'EOF'>~/ssl/kubernetes-csr.json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "10.11.40.69",
    "10.1.0.1",
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
      "OU": "System"
    }
  ]
}
EOF
```

#### 4.1.3 生成kubernetes证书和私钥:
```
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
```

#### 4.1.4 将生成的kubernetes证书和私钥分发到所有节点上:
```
scp kubernetes*.pem /opt/kubernetes/ssl/
```

#### 4.1.5 创建kube-apiserver使用的客户端token文件:
```
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
e2ad44a01412b9a1e14c8c6a0cfcdca7

cat <<'EOF'>/opt/kubernetes/ssl/bootstrap-token.csv
e2ad44a01412b9a1e14c8c6a0cfcdca7,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

#### 4.1.6 创建基础用户名/密码认证配置:
```
cat <<'EOF'>/opt/kubernetes/ssl/basic-auth.csv
admin,admin,1
readonly,readonly,2
EOF
```

#### 4.1.7 部署Kubernetes API Server:
```
cat <<'EOF'>/usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/opt/kubernetes/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
  --bind-address=10.11.40.69 \		\\需要修改地址;
  --insecure-bind-address=127.0.0.1 \
  --authorization-mode=Node,RBAC \
  --runtime-config=rbac.authorization.k8s.io/v1 \
  --kubelet-https=true \
  --anonymous-auth=false \
  --basic-auth-file=/opt/kubernetes/ssl/basic-auth.csv \
  --enable-bootstrap-token-auth \
  --token-auth-file=/opt/kubernetes/ssl/bootstrap-token.csv \
  --service-cluster-ip-range=10.1.0.0/16 \
  --service-node-port-range=80-40000 \
  --tls-cert-file=/opt/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/opt/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/opt/kubernetes/ssl/ca.pem \
  --service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/opt/kubernetes/ssl/ca.pem \
  --etcd-certfile=/opt/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/opt/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://10.11.40.71:2379,https://10.11.40.72:2379,https://10.11.40.73:2379 \		\\需要修改地址;
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/opt/kubernetes/log/api-audit.log \
  --event-ttl=1h \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

#### 4.1.8 启动API Server服务:
```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```

### 部署Controller Manager服务
### 4.2 部署Controller Manager服务
#### 4.2.1 生成Controller Manager服务:
```
cat <<'EOF'>/usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.1.0.0/16 \
  --cluster-cidr=10.2.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/opt/kubernetes/ssl/ca.pem \
  --leader-elect=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

#### 4.2.2 启动Controller Manager:
```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```

### 部署Kubernetes Scheduler:
### 4.3 部署Kubernetes Scheduler:
#### 4.3.1 生成kube-scheduler服务:
```
cat <<'EOF'>/usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/kubernetes/bin/kube-scheduler \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

#### 4.3.2 启动kube-scheduler服务:
```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```

### 部署kubectl命令行工具:
### 4.3 部署kubectl命令行工具
#### 4.3.1 创建kubectl命令行工具:
```
cd files/k8s-v1.10.3/bin
cp kubectl /opt/kubernetes/bin/

Kubernetes kubectl 命令自动补全:
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

#### 4.3.2 创建admin证书签名请求:
```
cat <<'EOF'>~/ssl/admin-csr.json
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
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
```

#### 4.3.3 生成admin证书和私钥:
```
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes admin-csr.json | cfssljson -bare admin

cp admin*.pem /opt/kubernetes/ssl/
```

#### 4.3.4 设置集群参数:
```
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://10.11.40.69:6443
```

#### 4.3.5 设置客户端认证参数:
```
kubectl config set-credentials admin \
   --client-certificate=/opt/kubernetes/ssl/admin.pem \
   --embed-certs=true \
   --client-key=/opt/kubernetes/ssl/admin-key.pem
```

#### 4.3.6 设置上下文参数:
```
kubectl config set-context kubernetes \
   --cluster=kubernetes \
   --user=admin
```

#### 4.3.7 设置默认上下文:
```
kubectl config use-context kubernetes
```

#### 4.3.8 使用kubectl工具:
```
kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}
```

## 5. Node节点部署
### 5.1 准备工作
#### 5.1.1 将命令分发到所有node节点上:
```
cd files/k8s-v1.10.3/bin
scp kubelet kube-proxy /opt/kubernetes/bin/

PS: 将内容分发到所有node节点上;
```

#### 5.1.2 创建角色绑定:
```
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

#### 5.1.3 创建kubelet bootstrapping kubeconfig文件设置集群参数:
```
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://10.11.40.69:6443 \
   --kubeconfig=bootstrap.kubeconfig
```

#### 5.1.4 设置客户端认证参数:
```
kubectl config set-credentials kubelet-bootstrap \
   --token=e2ad44a01412b9a1e14c8c6a0cfcdca7 \
   --kubeconfig=bootstrap.kubeconfig 
```

#### 5.1.5 设置上下文参数:
```
kubectl config set-context default \
   --cluster=kubernetes \
   --user=kubelet-bootstrap \
   --kubeconfig=bootstrap.kubeconfig
```

#### 5.1.6 选择默认上下文:
```
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

scp bootstrap.kubeconfig /opt/kubernetes/cfg

PS: 将内容分发到所有node节点上;
```

### 5.2 部署kubelet服务：
#### 5.2.1 设置CNI支持:
```
mkdir -p /etc/cni/net.d
cat <<'EOF'>/etc/cni/net.d/10-default.conf
{
        "name": "flannel",
        "type": "flannel",
        "delegate": {
            "bridge": "docker0",
            "isDefaultGateway": true,
            "mtu": 1400
        }
}
EOF
```

#### 5.2.2 创建kubelet目录:
```
mkdir -p /var/lib/kubelet
```

#### 5.2.3 创建kubelet服务配置:
```
示例:
k8s-node1:
cat <<'EOF'>/usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/opt/kubernetes/bin/kubelet \
  --address=10.11.40.71 \				\\需要修改地址;
  --hostname-override=10.11.40.71 \		\\需要修改地址;
  --pod-infra-container-image=mirrorgooglecontainers/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
  --kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
  --cert-dir=/opt/kubernetes/ssl \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/opt/kubernetes/bin/cni \
  --cluster-dns=10.1.0.2 \
  --cluster-domain=cluster.local. \
  --hairpin-mode hairpin-veth \
  --allow-privileged=true \
  --fail-swap-on=false \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log
Restart=on-failure
RestartSec=5
EOF
```

#### 5.2.4 启动Kubelet服务:
```
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```

#### 5.2.4 在k8s-master上执行查看csr请求:
```
kubectl get csr
```

#### 5.2.5 批准kubelet的TLS证书请求:
```
kubectl get csr|grep 'Pending' | awk 'NR>0{print $1}'| xargs kubectl certificate approve
```

### 5.3 部署Kubernetes Proxy服务：
#### 5.3.1 配置kube-proxy使用LVS:
```
yum install -y ipvsadm ipset conntrack
```

#### 5.3.2 创建kube-proxy证书请求:
```
cat <<'EOF'>~/ssl/kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "hosts": [],
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
      "OU": "System"
    }
  ]
}
EOF
```

#### 5.3.3 生成证书:
```
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

#### 5.3.4 分发证书到所有Node节点:
```
scp kube-proxy*.pem /opt/kubernetes/ssl/
```

#### 5.3.5 创建kube-proxy配置文件:
```
kubectl config set-cluster kubernetes \
   --certificate-authority=/opt/kubernetes/ssl/ca.pem \
   --embed-certs=true \
   --server=https://10.11.40.69:6443 \
   --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
   --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
   --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
   --embed-certs=true \
   --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
   --cluster=kubernetes \
   --user=kube-proxy \
   --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

#### 5.3.6 分发kubeconfig配置文件到所有Node节点:
```
scp kube-proxy.kubeconfig /opt/kubernetes/cfg/
```

#### 5.3.7 创建kube-proxy服务配置:
```
mkdir /var/lib/kube-proxy
示例:

k8s-node1:
cat <<'EOF'>/usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kubernetes/bin/kube-proxy \
  --bind-address=10.11.40.71 \			\\需要修改地址;
  --hostname-override=10.11.40.71 \		\\需要修改地址;
  --kubeconfig=/opt/kubernetes/cfg/kube-proxy.kubeconfig \
--masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --logtostderr=true \
  --v=2 \
  --logtostderr=false \
  --log-dir=/opt/kubernetes/log

Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

#### 5.3.8 启动Kubernetes Proxy:
```
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```

#### 5.3.9 检查LVS状态:
```
ipvsadm -L -n
```

#### 5.3.10 查看节点状态:
```
kubectl get node
NAME          STATUS    ROLES     AGE       VERSION
10.11.40.71   Ready     <none>    11d       v1.10.3
10.11.40.72   Ready     <none>    11d       v1.10.3
10.11.40.73   Ready     <none>    11d       v1.10.3
```

## 6. Flannel网络部署
### 6.1 准备工作:
#### 6.1.1 为Flannel生成证书配置文件:
```
cat <<'EOF'>~/ssl/flanneld-csr.json
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
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

#### 6.1.2 生成证书:
```
cfssl gencert -ca=/opt/kubernetes/ssl/ca.pem \
   -ca-key=/opt/kubernetes/ssl/ca-key.pem \
   -config=/opt/kubernetes/ssl/ca-config.json \
   -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
```

#### 6.1.3 分发证书到所有node节点上:
```
scp flanneld*.pem /opt/kubernetes/ssl/
```

#### 6.1.4 准备Flannel软件,将命令分发到所有node节点上:
```
cd files/flannel-v0.10.0-linux-amd64
scp flanneld mk-docker-opts.sh remove-docker0.sh /opt/kubernetes/bin/
```

#### 6.1.5 配置Flannel:
```
cat <<'EOF'>/opt/kubernetes/cfg/flannel
FLANNEL_ETCD="-etcd-endpoints=https://10.11.40.71:2379,https://10.11.40.72:2379,https://10.11.40.73:2379"	\\需要修改地址;
FLANNEL_ETCD_KEY="-etcd-prefix=/kubernetes/network"
FLANNEL_ETCD_CAFILE="--etcd-cafile=/opt/kubernetes/ssl/ca.pem"
FLANNEL_ETCD_CERTFILE="--etcd-certfile=/opt/kubernetes/ssl/flanneld.pem"
FLANNEL_ETCD_KEYFILE="--etcd-keyfile=/opt/kubernetes/ssl/flanneld-key.pem"
EOF
```

#### 6.1.6 设置Flannel系统服务:
```
cat <<'EOF'>/usr/lib/systemd/system/flannel.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
Before=docker.service

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/flannel
ExecStartPre=/opt/kubernetes/bin/remove-docker0.sh
ExecStart=/opt/kubernetes/bin/flanneld ${FLANNEL_ETCD} ${FLANNEL_ETCD_KEY} ${FLANNEL_ETCD_CAFILE} ${FLANNEL_ETCD_CERTFILE} ${FLANNEL_ETCD_KEYFILE}
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -d /run/flannel/docker

Type=notify

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
```

### 6.2 Flannel和CNI集成
#### 6.2.1 准备CNI插件,将命令分发到所有node节点上:
```
scp files/cni-plugins-amd64-v0.7.0 /opt/kubernetes/bin/cni
```

#### 6.2.2 创建Etcd的key (所有node节点操作):
```
etcdctl --ca-file /opt/kubernetes/ssl/ca.pem --cert-file /opt/kubernetes/ssl/flanneld.pem --key-file /opt/kubernetes/ssl/flanneld-key.pem \
      --no-sync -C https://10.11.40.71:2379,https://10.11.40.72:2379,https://10.11.40.73:2379 \		\\需要修改地址;
mk /kubernetes/network/config '{ "Network": "10.2.0.0/16", "Backend": { "Type": "vxlan", "VNI": 1 }}' >/dev/null 2>&1
```

#### 6.2.3 启动flannel服务:
```
chmod +x /opt/kubernetes/bin/*
systemctl daemon-reload
systemctl enable flannel
systemctl start flannel
systemctl status flannel
```

### 6.3 配置Docker使用Flannel:
#### 6.3.1 配置Docker使用Flannel (在所有节点都操作):
```
cat <<'EOF'>/usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service flannel.service
Wants=network-online.target
Requires=flannel.service

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd $DOCKER_OPTS
ExecReload=/bin/kill -s HUP $MAINPID
EnvironmentFile=-/etc/docker/daemon.json
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOF
```

#### 6.3.2 重启Docker服务:
```
systemctl daemon-reload
systemctl restart docker
systemctl status docker
```

## 7. 创建k8s应用
```
kubectl run net-test --image=alpine --replicas=3

查看创建状态:
kubectl get pod -o wide
```

## 8. 必备插件部署
### 8.1 CoreDNS插件部署:
```
cd files/addons/
kubectl apply -f coredns/

kubectl get pod -n kube-system
```

### 8.2 Heapster插件部署:
```
cd files/addons/
kubectl apply -f heapster/
```

### 8.3 Ingress插件部署:
```
cd files/addons/ingress-nginx-0.18.0

kubectl apply -f namespace.yaml 
kubectl apply -f default-backend.yaml 
kubectl apply -f configmap.yaml 
kubectl apply -f tcp-services-configmap.yaml
kubectl apply -f udp-services-configmap.yaml
kubectl apply -f rbac.yaml
kubectl apply -f with-rbac.yaml

创建规则访问:
例:
kubectl apply -f demo/alpha-ingress.yaml
```

### 8.4 Dashboard插件部署:
```
cd files/addons/
kubectl apply -f dashboard/

kubectl get pod -n kube-system|grep dashboard
```

## 访问Dashboard:
```
kubectl cluster-info

Kubernetes master is running at https://10.11.40.69:6443
CoreDNS is running at https://10.11.40.69:6443/api/v1/namespaces/kube-system/services/coredns:dns/proxy
kubernetes-dashboard is running at https://10.11.40.69:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
monitoring-grafana is running at https://10.11.40.69:6443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy


https://10.11.40.69:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
用户名:admin
密  码:admin
\\选择令牌模式登录;
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```