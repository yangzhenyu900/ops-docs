# kubeadm部署Kubernetes:
- 部署Kubernetes-v1.11.2版本。

## 版本明细: Release-v1.11.2
- OS:           CentOS Linux release 7.4.1708 (Core)
- kernel:       3.10.0-693.el7.x86_64
- Kubernetes：  v1.11.2
- Docker:       18.06.0-ce
- Flannel:      v0.10.0
- kubeadm:      v1.11.2

## 部署准备: 至少三个节点，请配置好主机名解析(必备)

## 1. 系统初始化 (这里不在演示操作):
1. 规范主机名;
2. 设置/etc/hosts主机名内部解析;
3. 关闭SELinux和防火墙;
4. ssh密钥连接;
5. 设置时间同步;


### 1.1 机器分配情况:
```
/etc/hosts:
192.168.2.2    k8s-master     k8s-master.myunsheji.com
192.168.2.3    k8s-node1      k8s-node1.myunsheji.com
192.168.2.4    k8s-node2      k8s-node2.myunsheji.com
```

### 1.2 准备环境 (所有节点操作):

#### 1) 配置docker和kubernetes国内源:
```
Docker源:
curl -o /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

Kubernetes源:
cat <<'EOF'>/etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 2) 安装docker及其它软件:
```
yum install -y docker-ce-18.06.0.ce-3.el7.x86_64 kubelet kubeadm kubectl
```

#### 3) 设置私有仓库和加速器:
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

#### 4) 启动docker服务:
```
systemctl daemon-reload
systemctl enable docker kubelet
systemctl start docker
systemctl status docker
```

#### 5) 登录docker私有仓库方式:
```
echo "rqKwWsB+Y1HlAdByT3bT"|docker login -u docker-dev --password-stdin 10.11.40.59:8082
```

#### 6) Kubernetes kubectl命令自动补全:
```
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

## 2. 修改kubelet配置文件,忽略错误 (所有节点都操作):
```
[root@k8s-master ~]# cat <<'EOF'>/etc/sysconfig/kubelet 
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
EOF

[root@k8s-master ~]# echo "net.bridge.bridge-nf-call-iptables = 1" >>/etc/sysctl.conf 
[root@k8s-master ~]# sysctl -p
```

## 3. 使用kubeadm工具进行初始化操作 (Master操作):
### 3.1 获取镜像:
```
从私有仓库拉取镜像:
docker pull 10.11.40.59:8082/kube-apiserver-amd64:v1.11.2
docker pull 10.11.40.59:8082/kube-controller-manager-amd64:v1.11.2
docker pull 10.11.40.59:8082/kube-scheduler-amd64:v1.11.2
docker pull 10.11.40.59:8082/kube-proxy-amd64:v1.11.2
docker pull 10.11.40.59:8082/pause:3.1
docker pull 10.11.40.59:8082/etcd-amd64:3.2.18
docker pull 10.11.40.59:8082/coredns:1.1.3


为镜像打tag,否则kubeadm初始化时识别不到带有私有仓库地址的镜像(还未找到修改镜像名称的位置):
docker tag 10.11.40.59:8082/kube-apiserver-amd64:v1.11.2	  k8s.gcr.io/kube-apiserver-amd64:v1.11.2
docker tag 10.11.40.59:8082/kube-controller-manager-amd64:v1.11.2 k8s.gcr.io/kube-controller-manager-amd64:v1.11.2
docker tag 10.11.40.59:8082/kube-scheduler-amd64:v1.11.2	  k8s.gcr.io/kube-scheduler-amd64:v1.11.2
docker tag 10.11.40.59:8082/kube-proxy-amd64:v1.11.2		  k8s.gcr.io/kube-proxy-amd64:v1.11.2
docker tag 10.11.40.59:8082/pause:3.1				  k8s.gcr.io/pause:3.1
docker tag 10.11.40.59:8082/etcd-amd64:3.2.18			  k8s.gcr.io/etcd-amd64:3.2.18
docker tag 10.11.40.59:8082/coredns:1.1.3			  k8s.gcr.io/coredns:1.1.3


清理镜像:
docker rmi $(docker images|grep 10.11.40.59:8082|awk '{print $1":"$2}')


PS: 执行初始化操作会去获取一些镜像,因网络问题获取镜像会存在问题,因此先拉取下来修改tag进行导入(这里已经把镜像放在了私有仓库中);
```

### 3.2 使用kubeadm进行初始化操作:
```
[root@k8s-master ~]# kubeadm init --kubernetes-version=v1.11.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12 --ignore-preflight-errors=Swap


初始化完毕提示操作信息:
To start using your cluster, you need to run the following as a regular user:
\\需要操作的内容:
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:
\\节点加入集群操作,需要记录下来,忘记获取比较麻烦:
  kubeadm join 192.168.2.2:6443 --token 35ok5a.3ycwtlmca8az8a7g --discovery-token-ca-cert-hash sha256:c462274c03bbe8e1f8102631108590e0650bb2164bb16a343e45723c2749f628
```

### 3.3 操作上述提示步骤(所有节点都操作):
```
操作步骤:
[root@k8s-master ~]# mkdir -p $HOME/.kube
[root@k8s-master ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@k8s-master ~]# chown $(id -u):$(id -g) $HOME/.kube/config
[root@k8s-master ~]# kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
[root@k8s-master ~]# kubectl get nodes	\\节点不正常,是因为缺少网络组件;
NAME         STATUS     ROLES     AGE       VERSION
k8s-master   NotReady   master    9m        v1.11.2
```

## 4. 部署网络组件flannel (所有节点都操作):

### 4.1 获取镜像:
```
获取地址:
https://github.com/coreos/flannel

从私有仓库拉取镜像:
docker pull 10.11.40.59:8082/flannel:v0.10.0-amd64
docker pull 10.11.40.59:8082/flannel:v0.10.0-arm64
docker pull 10.11.40.59:8082/flannel:v0.10.0-arm
docker pull 10.11.40.59:8082/flannel:v0.10.0-ppc64le
docker pull 10.11.40.59:8082/flannel:v0.10.0-s390x


为镜像打tag,否则需要修改yaml文件镜像名称:
docker tag 10.11.40.59:8082/flannel:v0.10.0-amd64   quay.io/coreos/flannel:v0.10.0-amd64
docker tag 10.11.40.59:8082/flannel:v0.10.0-arm64   quay.io/coreos/flannel:v0.10.0-arm64
docker tag 10.11.40.59:8082/flannel:v0.10.0-arm     quay.io/coreos/flannel:v0.10.0-arm
docker tag 10.11.40.59:8082/flannel:v0.10.0-ppc64le quay.io/coreos/flannel:v0.10.0-ppc64le
docker tag 10.11.40.59:8082/flannel:v0.10.0-s390x   quay.io/coreos/flannel:v0.10.0-s390x


清理镜像:
docker rmi $(docker images|grep 10.11.40.59:8082|awk '{print $1":"$2}')


PS: 执行初始化操作会去获取一些镜像,因网络问题获取镜像会存在问题,因此先拉取下来修改tag进行导入(这里已经把镜像放在了私有仓库中);
```

### 4.2 部署flannel:
```
Kubernetes v1.7+版本直接运行如下命令:
[root@k8s-master ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 4.3 查看运行状态:
```
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS    ROLES     AGE       VERSION
k8s-master   Ready     master    52m       v1.11.2

[root@k8s-master ~]# kubectl get pods -n kube-system
NAME                                 READY     STATUS    RESTARTS   AGE
coredns-78fcdf6894-92bsz             1/1       Running   0          54m
coredns-78fcdf6894-mkmr6             1/1       Running   0          54m
etcd-k8s-master                      1/1       Running   0          1m
kube-apiserver-k8s-master            1/1       Running   0          1m
kube-controller-manager-k8s-master   1/1       Running   0          1m
kube-flannel-ds-amd64-86pxh          1/1       Running   0          1m
kube-proxy-htvcz                     1/1       Running   0          54m
kube-scheduler-k8s-master            1/1       Running   0          1m
```

## 5. 将节点加入集群 (所有node节点操作):
```
kubeadm join 192.168.2.2:6443 --token 35ok5a.3ycwtlmca8az8a7g --discovery-token-ca-cert-hash sha256:c462274c03bbe8e1f8102631108590e0650bb2164bb16a343e45723c2749f628 --ignore-preflight-errors=Swap
```

## 6. 在master上查看节点加入信息:
```
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS     ROLES     AGE       VERSION
k8s-master   Ready      master    1h        v1.11.2
k8s-node1    NotReady   <none>    22s       v1.11.2
k8s-node2    NotReady   <none>    4s        v1.11.2


等待一会,再次查看:
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS    ROLES     AGE       VERSION
k8s-master   Ready     master    1h        v1.11.2
k8s-node1    Ready     <none>    28m       v1.11.2
k8s-node2    Ready     <none>    28m       v1.11.2

PS: Master节点虽然部署了node节点服务,但是它不会被调度去运行容器应用;
```

## 7. 部署一个应用测试环境:
```
[root@k8s-master ~]# kubectl run nginx-deploy --image=nginx:1.14-alpine --port=80 --replicas=1
deployment.apps/nginx-deploy created
[root@k8s-master ~]# kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
nginx-deploy-5b595999-m9xw2   1/1       Running   0          37s
[root@k8s-master ~]# kubectl get deployment
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   1         1         1            1           45s
[root@k8s-master ~]# kubectl get pods -o wide
NAME                          READY     STATUS    RESTARTS   AGE       IP           NODE        NOMINATED NODE
nginx-deploy-5b595999-m9xw2   1/1       Running   0          1m        10.244.2.2   k8s-node2   <none>
```
