# 使用 kubeadm 安装 k8s 集群


## 使用 kubeadm 在 CentOS 8.3 上安装 k8s 集群

### 清单

#### 服务器

|**IP**|主机名|**配置**|节点类别|
| :-----: | :-----: | :-----: | :-----: |
|10.9.56.239|master1-10-9-56-239|2核/4G/200G磁盘|master|
|10.9.158.227|master2-10-9-158-227|2核/4G/200G磁盘|master|
|10.9.153.152|master3-10-9-153-152|2核/4G/200G磁盘|master|
|10.9.167.63|node-10-9-167-63|2核/4G/200G磁盘|worker|
|10.9.32.181|node-10-9-32-181|2核/4G/200G磁盘|worker|
|10.9.63.155|node-10-9-63-155|2核/4G/200G磁盘|worker|

#### 软件版本

|**软件**|**版本**|
| :-----: | :-----: |
|系统|CentOS 8.3|
|Kubernetes|v1.27.3|
|Docker|20.10.21|
|Keepalived|2.0.17|
|Haproxy|2.1.4|

### 基础环境配置
>
> 如无特殊说明，所有节点均需要执行操作

#### 确保每个节点上 MAC 地址和 product\_uuid 的唯一性

```bash
# 检测 mac 地址
ip link
ifconfig -a

# 检查 product_uuid
cat /sys/class/dmi/id/product_uuid
```

#### 修改服务器 hostname

```bash
# 在 10.9.56.239 上执行
hostnamectl set-hostname master1-10-9-56-239
# 在 10.9.158.227 上执行
hostnamectl set-hostname master2-10-9-158-227
# 在 10.9.153.152 上执行
hostnamectl set-hostname master3-10-9-153-152
```

退出，重新登录，可以看到主机名生效。

#### 配置服务器 hosts
>
> 1. 映射关系对 `10.9.56.239 cluster-endpoint` 与 `kubeadm init` 时传递参数 `--control-plane-endpoint=cluster-endpoint` 关联，配置高可用集群时，该参数必须指定。
> 2. 高可用集群部署成功后，可将该映射 IP 地址改为负载均衡地址。
> 3. 该映射 IP 为第一个部署的 master 节点地址。

```bash
cat >> /etc/hosts<<EOF
10.9.56.239 master1-10-9-56-239
10.9.158.227 master2-10-9-158-227
10.9.153.152 master3-10-9-153-152
10.9.56.239 cluster-endpoint
EOF
```

#### 配置 ssh 互信

```bash
# 生成 ssh 公私钥
ssh-keygen

# 配置 authorized_keys（各节点交叉执行）
ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.9.56.239
ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.9.158.227
ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.9.153.152
```

#### 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
```

#### 关闭 swap，注释 swap 分区

```bash
# 临时有效
swapoff -a
# 永久生效，修改 /etc/fstab，删除如下行
/dev/mapper/cl-swap   swap  swap  defaults  0 0
```

#### 禁用 SELinux

```bash
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

#### 配置内核参数，转发 IPv4 并让 iptables 看到桥接流量
>
> 参考：[k8s 容器运行时转发 IPv4 并让 iptables 看到桥接流量](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#%E8%BD%AC%E5%8F%91-ipv4-%E5%B9%B6%E8%AE%A9-iptables-%E7%9C%8B%E5%88%B0%E6%A1%A5%E6%8E%A5%E6%B5%81%E9%87%8F)

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

#### 添加阿里源

```bash
# 配置yum源
cd /etc/yum.repos.d ; mkdir bak; mv CentOS-Base.repo bak/; mv CentOS-Linux-* bak/
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo
```

#### 安装常用包

```bash
yum install -y vim bash-completion net-tools gcc
```

#### 设置时间同步

```bash
yum install -y chrony

# 同步的时间服务器修改为国家授时中心 NTP 服务器
sed -i 's/^pool 2.centos.pool.ntp.org iburst$/pool ntp.ntsc.ac.cn iburst/' /etc/chrony.conf

systemctl start chronyd
systemctl enable chronyd

# 查看同步的时间服务器
chronyc sources
```

#### 安装 docker

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum -y install docker-ce
```

* 安装 docker 如果报错：

```bash
[root@master1-10-9-56-239 ~]# yum install docker-ce
Repository cr is listed more than once in the configuration
Repository extras is listed more than once in the configuration
Repository extras-source is listed more than once in the configuration
Repository fasttrack is listed more than once in the configuration
Docker CE Stable - x86_64                                                                                              66 kB/s |  29 kB     00:00
Error:
 Problem: problem with installed package buildah-1.22.3-2.module_el8.5.0+911+f19012f9.x86_64
  - package buildah-1.22.3-2.module_el8.5.0+911+f19012f9.x86_64 requires runc >= 1.0.0-26, but none of the providers can be installed
  - package containerd.io-1.4.3-3.1.el8.x86_64 conflicts with runc provided by runc-1.0.2-1.module_el8.5.0+911+f19012f9.x86_64
...
  - cannot install the best candidate for the job
  - package runc-1.0.0-56.rc5.dev.git2abd837.module_el8.3.0+569+1bada2e4.x86_64 is filtered out by modular filtering
  - package runc-1.0.0-66.rc10.module_el8.5.0+1004+c00a74f5.x86_64 is filtered out by modular filtering
  - package runc-1.0.0-72.rc92.module_el8.5.0+1006+8d0e68a2.x86_64 is filtered out by modular filtering
(try to add '--allowerasing' to command line to replace conflicting packages or '--skip-broken' to skip uninstallable packages or '--nobest' to use not only best candidate packages)
```

* 解决办法

```bash
# 删除 podman、buildah
yum remove -y podman buildah containers-common

# 再重新执行 docker 安装命令
yum -y install docker-ce
```

启动 docker

```bash
systemctl start docker
systemctl enable docker
```

添加阿里云 docker 仓库加速器：

```bash
cat >/etc/docker/daemon.json<<EOF
{
  "registry-mirrors": ["https://fl791z1h.mirror.aliyuncs.com"]
}
EOF

systemctl reload docker
systemctl status docker containerd
```

> 安装 docker 后，默认会自动安装容器运行时 containerd。kubernetes 也支持使用 containerd 作为容器运行时。

#### 重载沙箱（pause）镜像源为阿里镜像源
>
> 参考：[k8s 容器运行时重载沙箱（pause）镜像](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#override-pause-image-containerd)

```bash
# 导出默认配置，config.toml 这个文件默认是不存在的
containerd config default > /etc/containerd/config.toml
grep sandbox_image /etc/containerd/config.toml

# 按上一步输出选择
sed -i "s#k8s.gcr.io/pause#registry.aliyuncs.com/google_containers/pause#g" /etc/containerd/config.toml
或
sed -i "s#registry.k8s.io/pause#registry.aliyuncs.com/google_containers/pause#g" /etc/containerd/config.toml

# 确认修改成功
grep sandbox_image  /etc/containerd/config.toml
```

#### 配置 systemd cgroup 驱动
>
> 参考：[k8s 容器运行时配置 systemd cgroup 驱动](https://kubernetes.io/zh-cn/docs/setup/production-environment/container-runtimes/#containerd-systemd)

```bash
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
# 应用所有更改后，重新启动containerd
systemctl restart containerd
```

#### 安装 kubectl、kubelet、kubeadm

> 参考：[k8s kubeadm 安装](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)

##### 配置 k8s yum 源

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

##### 安装

```bash
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```

> ⚠️**注意**：kubelet 现在每隔几秒就会重启，属于正常现象，因为它陷入了一个等待 kubeadm 指令的死循环。

#### 设置 kubernetes 的容器运行时为 containerd
>
> 生成的配置在`/etc/crictl.yaml`，可以随时修改。

```bash
crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```

### 使用 kubeadm 创建集群
>
> 参考：[使用 kubeadm 创建集群](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

#### 部署 master1 节点

##### master1 上执行 kubeadm init

```bash
# –-apiserver-advertise-address 用于为控制平面节点的 API server 设置广播地址。
# –-image-repository 指定从什么位置来拉取镜像。
# --control-plane-endpoint 用于为所有控制平面节点设置共享端点。如果不设置，则无法将单个控制平面 kubeadm 集群升级成高可用。
# --upload-certs 指定将在所有控制平面实例之间的共享证书上传到集群。后面 join 其他 master 节点时，可以使用该证书。
# –-kubernetes-version 指定 k8s 版本号。
# –-pod-network-cidr 指定 Pod 网络的范围。不同 CNI 默认网段也不一样，Calico 默认为 192.168.0.0/16。

kubeadm init \
--apiserver-advertise-address=10.9.56.239 \
--image-repository registry.aliyuncs.com/google_containers \
--control-plane-endpoint=cluster-endpoint \
--upload-certs \
--kubernetes-version v1.27.3 \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=192.168.0.0/16 \
--v=5
```

部署成功后，输出如下：

```bash
...
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

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join cluster-endpoint:6443 --token 89si13.zc0n2y0d53td5wgs \
 --discovery-token-ca-cert-hash sha256:3d7be1719e14d6d1273a8184122093d0e6666174f2f62f890018fda618251bc5 \
 --control-plane \
 --certificate-key e467406274bac7ea4db433835f4b825d1f3614abe2be7d584965a5183c0b23e2

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join cluster-endpoint:6443 --token 89si13.zc0n2y0d53td5wgs \
 --discovery-token-ca-cert-hash sha256:3d7be1719e14d6d1273a8184122093d0e6666174f2f62f890018fda618251bc5
```

##### master1 配置 kube config 环境变量

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 设置环境变量
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```

##### 查看 node 信息，状态为 `NotReady`

```bash
[root@master1-10-9-56-239 ~]# kubectl get nodes
NAME      STATUS     ROLES           AGE     VERSION
master3   NotReady   control-plane   2m50s   v1.25.2
```

* 查看 kubelet 服务日志：

```bash
journalctl -xeu kubelet
```

* 有如下报错：

```bash
Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
```

需要先安装 CNI 插件

##### 安装 pod 网络插件 CNI：Calico
>
> 参考：[Install Calico](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart#how-to)

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.3/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.3/manifests/custom-resources.yaml
```

##### 查看 pods、nodes 信息，状态均正常

```bash
[root@master1-10-9-56-239 ~]# kubectl get pods -A
calico-apiserver   calico-apiserver-bc5b75576-97k76               1/1     Running   0             11h
calico-apiserver   calico-apiserver-bc5b75576-xvm26               1/1     Running   0             11h
calico-system      calico-kube-controllers-5cb6bfc9bd-hcbk2       1/1     Running   0             11h
calico-system      calico-node-n44pc                              0/1     Running   0             11h
calico-system      calico-typha-7cdd94c5d7-22xz8                  1/1     Running   0             11h
kube-system        coredns-7bdc4cb885-6jsjg                       1/1     Running   0             11h
kube-system        coredns-7bdc4cb885-wr2jh                       1/1     Running   0             11h
kube-system        etcd-master1-10-9-125-21                       1/1     Running   0             11h
kube-system        kube-apiserver-master1-10-9-56-239             1/1     Running   0             11h
kube-system        kube-controller-manager-master1-10-9-56-239    1/1     Running   1 (11h ago)   11h
kube-system        kube-proxy-94cjw                               1/1     Running   0             11h
kube-system        kube-scheduler-master1-10-9-125-21             1/1     Running   1 (11h ago)   11h
tigera-operator    tigera-operator-5c5bbf4f78-n6cwb               1/1     Running   2 (11h ago)   11h

[root@master1-10-9-56-239 ~]# kubectl get nodes
NAME                  STATUS   ROLES           AGE   VERSION
master1-10-9-56-239   Ready    control-plane   14m   v1.27.3
```

##### Calico 安装问题

> calico-node pod Running，but not ready

有两种方法解决：

1. 使用 kubectl edit 直接修改 Calico CRD 资源配置
2. 下载 Calico CRD 资源文件再修改配置（`推荐`）

###### kubectl edit 修改资源配置

* 查看 pod `calico-node-n44pc` 详情，报错：`calico/node is not ready: BIRD is not ready: BGP not established`

```bash
$> kubectl describe -n calico-system pod calico-node-8dl5h
...
QoS Class:                   BestEffort
Node-Selectors:              kubernetes.io/os=linux
Tolerations:                 :NoSchedule op=Exists
                             :NoExecute op=Exists
                             CriticalAddonsOnly op=Exists
                             node.kubernetes.io/disk-pressure:NoSchedule op=Exists
                             node.kubernetes.io/memory-pressure:NoSchedule op=Exists
                             node.kubernetes.io/network-unavailable:NoSchedule op=Exists
                             node.kubernetes.io/not-ready:NoExecute op=Exists
                             node.kubernetes.io/pid-pressure:NoSchedule op=Exists
                             node.kubernetes.io/unreachable:NoExecute op=Exists
                             node.kubernetes.io/unschedulable:NoSchedule op=Exists
Events:
  Type     Reason     Age                     From     Message
  ----     ------     ----                    ----     -------
  Warning  Unhealthy  4m54s (x4808 over 11h)  kubelet  (combined from similar events): Readiness probe failed: 2023-07-01 01:49:28.765 [INFO][66196] confd/health.go 180: Number of node(s) with BGP peering established = 0
calico/node is not ready: BIRD is not ready: BGP not established with 192.168.21.0,192.168.178.64
```

* 安装 calicoctl 工具

> 参考：[install calicoctl](https://docs.tigera.io/calico/latest/operations/calicoctl/install#install-calicoctl-as-a-binary-on-a-single-host)

```bash
curl -L https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64 -o calicoctl
chmod +x ./calicoctl
```

* 查看 calico 节点状态

```bash
$> calicoctl node status
Calico process is running.

IPv4 BGP status
+----------------+-------------------+-------+----------+--------------------------------+
|  PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |              INFO              |
+----------------+-------------------+-------+----------+--------------------------------+
| 192.168.21.0   | node-to-node mesh | start | 14:02:16 | Active Socket: Connection      |
|                |                   |       |          | closed                         |
| 192.168.178.64 | node-to-node mesh | start | 14:05:31 | Active Socket: Connection      |
|                |                   |       |          | closed                         |
+----------------+-------------------+-------+----------+--------------------------------+

IPv6 BGP status
No IPv6 peers found.
```

注意到 `PEER ADDRESS` 不是节点 IP，导致请求失败。Calico 自动检测 IP 地址默认使用 `first-found` 方法，获得错误地址，需要手动指定检测方法。

* 手动修改 calico 自定义资源配置 `nodeAddressAutodetectionV4` 为节点网卡名称。

> 参考：[calico IP autodetection](https://docs.tigera.io/calico/latest/networking/ipam/ip-autodetection)

```bash
$> kubectl edit Installation default
...
spec:
  calicoNetwork:
    ...
    nodeAddressAutodetectionV4:
      interface: eth0
...
```

###### 下载 Calico CRD 资源文件再修改配置

* 下载 Calico 资源文件

```shell
wget https://raw.githubusercontent.com/projectcalico/calico/v3.24.3/manifests/custom-resources.yaml
```

* 增加网卡配置

```shell
$> vim custom-resources.yaml
```

增加如下配置：

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    ...
    nodeAddressAutodetectionV4:
      interface: eth0
```

* 执行部署文件

```shell
kubectl apply -f custom-resources.yaml
```

### 使用 kubeadm 创建高可用集群
>
> 参考：[利用 kubeadm 创建高可用集群](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/high-availability/)

* **说明1**：master1 节点上执行 `kubeadm init` 成功后会输出加入其他节点的 join 命令，如：

```bash
...
You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join cluster-endpoint:6443 --token 89si13.zc0n2y0d53td5wgs \
 --discovery-token-ca-cert-hash sha256:3d7be1719e14d6d1273a8184122093d0e6666174f2f62f890018fda618251bc5 \
 --control-plane \
 --certificate-key e467406274bac7ea4db433835f4b825d1f3614abe2be7d584965a5183c0b23e2

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join cluster-endpoint:6443 --token 89si13.zc0n2y0d53td5wgs \
 --discovery-token-ca-cert-hash sha256:3d7be1719e14d6d1273a8184122093d0e6666174f2f62f890018fda618251bc5
```

* **说明2**：kubeadm-certs Secret（证书）和解密密钥会在两个小时后失效，如果要重新上传证书并生成新的解密密钥，请在**已加入集群节点的控制平面上（如 master1）**使用以下命令：

```bash
[root@master1-10-9-56-239 ~]# kubeadm init phase upload-certs --upload-certs
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
e467406274bac7ea4db433835f4b825d1f3614abe2be7d584965a5183c0b23e2
```

* **说明3**：如果 join 命令未保存，可以执行如下命令重新生成：

> **注意**
>
> 1. 生成的命令无 `--certificate-key` 、`--control-plane` 参数，可以执行 `kubeadm init phase upload-certs --upload-certs` 后，自行加上 `--certificate-key`。
> 2. 如果加入的是 master 节点到集群，则需要指定 `--control-plane` 。

```bash
[root@master1-10-9-56-239 ~]# kubeadm token create --print-join-command
kubeadm join cluster-endpoint:6443 --token udh68j.az1v5k24b7byp6bj --discovery-token-ca-cert-hash sha256:3d7be1719e14d6d1273a8184122093d0e6666174f2f62f890018fda618251bc5
```

#### 部署 master2 节点

##### master2 上执行 join 命令加入集群

```bash
# 在 master2 节点上执行
kubeadm join cluster-endpoint:6443 --token udh68j.az1v5k24b7byp6bj \
 --discovery-token-ca-cert-hash sha256:3d7be1719e14d6d1273a8184122093d0e6666174f2f62f890018fda618251bc5 \
 --control-plane \
 --certificate-key e467406274bac7ea4db433835f4b825d1f3614abe2be7d584965a5183c0b23e2
```

部署成功后，输出如下：

```bash
...
This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

 mkdir -p $HOME/.kube
 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

##### master2 配置 kube config 环境变量

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 设置环境变量
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```

##### master2 确认 pods、nodes 状态

```bash
[root@master2-10-9-158-227 ~]# kubectl get pods -A
NAMESPACE      NAME                                           READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-gqnxz                          1/1     Running   0          33m
kube-flannel   kube-flannel-ds-wxxns                          1/1     Running   0          21h
kube-system    coredns-c676cc86f-2kc85                        1/1     Running   0          21h
kube-system    coredns-c676cc86f-qxq6b                        1/1     Running   0          21h
kube-system    etcd-master1-10-9-56-239                       1/1     Running   0          21h
kube-system    etcd-master2-10-9-158-227                      1/1     Running   0          32m
kube-system    kube-apiserver-master1-10-9-56-239             1/1     Running   0          21h
kube-system    kube-apiserver-master2-10-9-158-227            1/1     Running   0          32m
kube-system    kube-controller-manager-master1-10-9-56-239    1/1     Running   0          21h
kube-system    kube-controller-manager-master2-10-9-158-227   1/1     Running   0          31m
kube-system    kube-proxy-czmhv                               1/1     Running   0          21h
kube-system    kube-proxy-fmw87                               1/1     Running   0          33m
kube-system    kube-scheduler-master1-10-9-56-239             1/1     Running   0          21h
kube-system    kube-scheduler-master2-10-9-158-227            1/1     Running   0          32m

[root@master2-10-9-158-227 ~]# kubectl get nodes
NAME                   STATUS   ROLES           AGE   VERSION
master1-10-9-56-239    Ready    control-plane   21h   v1.27.3
master2-10-9-158-227   Ready    control-plane   30m   v1.27.3
```

#### 部署 master3 节点

##### master3 上执行 join 命令加入集群

```bash
# 在 master3 节点上执行
kubeadm join cluster-endpoint:6443 --token 7dfoti.tcwf1vch4j424qdc \
 --discovery-token-ca-cert-hash sha256:3d7be1719e14d6d1273a8184122093d0e6666174f2f62f890018fda618251bc5 \
 --control-plane \
 --certificate-key e467406274bac7ea4db433835f4b825d1f3614abe2be7d584965a5183c0b23e2
```

部署成功后，输出如下：

```bash
...
This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

 mkdir -p $HOME/.kube
 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

##### master3 配置 kube config 环境变量

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 设置环境变量
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source ~/.bash_profile
```

##### master3 确认 pods、nodes 状态

```bash
[root@master3-10-9-153-152 ~]# kubectl get pods -A
NAMESPACE      NAME                                           READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-6zctk                          1/1     Running   0          100s
kube-flannel   kube-flannel-ds-gqnxz                          1/1     Running   0          46m
kube-flannel   kube-flannel-ds-wxxns                          1/1     Running   0          21h
kube-system    coredns-c676cc86f-2kc85                        1/1     Running   0          22h
kube-system    coredns-c676cc86f-qxq6b                        1/1     Running   0          22h
kube-system    etcd-master1-10-9-56-239                       1/1     Running   0          22h
kube-system    etcd-master2-10-9-158-227                      1/1     Running   0          46m
kube-system    etcd-master3-10-9-153-152                      1/1     Running   0          90s
kube-system    kube-apiserver-master1-10-9-56-239             1/1     Running   0          22h
kube-system    kube-apiserver-master2-10-9-158-227            1/1     Running   0          46m
kube-system    kube-apiserver-master3-10-9-153-152            1/1     Running   0          84s
kube-system    kube-controller-manager-master1-10-9-56-239    1/1     Running   0          22h
kube-system    kube-controller-manager-master2-10-9-158-227   1/1     Running   0          45m
kube-system    kube-controller-manager-master3-10-9-153-152   1/1     Running   0          8s
kube-system    kube-proxy-czmhv                               1/1     Running   0          22h
kube-system    kube-proxy-fmw87                               1/1     Running   0          46m
kube-system    kube-proxy-vcgj6                               1/1     Running   0          100s
kube-system    kube-scheduler-master1-10-9-56-239             1/1     Running   0          22h
kube-system    kube-scheduler-master2-10-9-158-227            1/1     Running   0          46m
kube-system    kube-scheduler-master3-10-9-153-152            1/1     Running   0          16s

[root@master3-10-9-153-152 ~]# kubectl get nodes
NAME                   STATUS   ROLES           AGE   VERSION
master1-10-9-56-239    Ready    control-plane   22h   v1.27.3
master2-10-9-158-227   Ready    control-plane   46m   v1.27.3
master3-10-9-153-152   Ready    control-plane   93s   v1.27.3
```

### 配置 kube-apiserver 高可用（所有master 节点上执行）
>
> 参考：[软件负载平衡选项指南](https://git.k8s.io/kubeadm/docs/ha-considerations.md#options-for-software-load-balancing)

选择基于 `keepalived + haproxy` 为 `kube-apiserver` 创建负载均衡器，并通过 k8s pod 的方式部署 keepalived 和 haproxy。

首先需要选定一个 VIP，本示例中 VIP 为：`10.9.37.82` 。

> 如果为云上环境，可以在 VPC 下单独申请 VIP。

#### 初始化 keepalived 配置文件

* keepalived 配置文件 `/etc/keepalived/keepalived.conf`：

> **修改配置：**
>
> 1. state：选择第一个节点为 MASTER，其他均为 BACKUP
> 2. interface：即网卡，如 eth0
> 3. priority：优先级，MASTER 节点为 101，BACKUP 节点为 100

```bash
mkdir /etc/keepalived && cat >> /etc/keepalived/keepalived.conf<<EOF
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state MASTER
    # state BACKUP
    interface eth0
    virtual_router_id 51
    priority 101
    # priority 100
    authentication {
        auth_type PASS
        auth_pass 42
    }
    virtual_ipaddress {
        10.9.37.82/16
    }
    track_script {
        check_apiserver
    }
}
EOF
```

* keepalived 健康检测脚本`/etc/keepalived/check_apiserver.sh`：

> 按实际修改 VIP 地址，如 `10.9.37.82`

```bash
#!/bin/sh

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

vip=10.9.37.82

curl --silent --max-time 2 --insecure https://localhost:8443/ -o /dev/null || errorExit "Error GET https://localhost:8443/"
if ip addr | grep -q ${vip}; then
    curl --silent --max-time 2 --insecure https://${vip}:8443/ -o /dev/null || errorExit "Error GET https://${vip}:8443/"
fi
```

#### 初始化 haproxy 配置文件

* haproxy 配置文件`/etc/haproxy/haproxy.cfg`：

> **修改配置：**
>
> 1. 修改 backend 配置中各 master 节点 IP 地址

```bash
mkdir /etc/haproxy && cat >> /etc/haproxy/haproxy.cfg<<EOF
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          20s
    timeout server          20s
    timeout http-keep-alive 10s
    timeout check           10s

#---------------------------------------------------------------------
# apiserver frontend which proxys to the control plane nodes
#---------------------------------------------------------------------
frontend apiserver
    bind *:8443
    mode tcp
    option tcplog
    default_backend apiserver

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode    tcp
    option  ssl-hello-chk
    balance roundrobin
    server  k8s-master1 10.9.56.239:6443 check
    server  k8s-master2 10.9.158.227:6443 check
    server  k8s-master3 10.9.153.152:6443 check
EOF
```

#### 生成 Pod 清单文件

将 `keepalived、haproxy` Pod 清单文件放入 `/etc/kubernetes/manifests` 目录后，**k8s 会监听目录文件变化，自动拉起 keepalived、haproxy Pod**。

* keepalived

```bash
cat >> /etc/kubernetes/manifests/keepalived.yaml<<EOF
apiVersion: v1
kind: Pod
metadata:
  name: keepalived
  namespace: kube-system
spec:
  containers:
  - image: osixia/keepalived:2.0.17
    name: keepalived
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_BROADCAST
        - NET_RAW
    volumeMounts:
    - mountPath: /usr/local/etc/keepalived/keepalived.conf
      name: config
    - mountPath: /etc/keepalived/check_apiserver.sh
      name: check
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/keepalived/keepalived.conf
    name: config
  - hostPath:
      path: /etc/keepalived/check_apiserver.sh
    name: check
EOF
```

* haproxy

```bash
cat >> /etc/kubernetes/manifests/haproxy.yaml<<EOF
apiVersion: v1
kind: Pod
metadata:
  name: haproxy
  namespace: kube-system
spec:
  containers:
  - image: haproxy:2.1.4
    name: haproxy
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: localhost
        path: /healthz
        port: 8443
        scheme: HTTPS
    volumeMounts:
    - mountPath: /usr/local/etc/haproxy/haproxy.cfg
      name: haproxyconf
      readOnly: true
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/haproxy/haproxy.cfg
      type: FileOrCreate
    name: haproxyconf
EOF
```

#### 修改 /ect/hosts 中 cluster-endpoint 映射地址

```bash
# 将 cluster-endpoint 映射 IP 改为 VIP
[root@master1-10-9-56-239 ~]# cat /etc/hosts
...
10.9.37.82 cluster-endpoint
```

#### 验证 vip 配置正常

```yaml
$> kubectl get pods -A -v 10
I0701 11:07:45.814632  577407 round_trippers.go:466] curl -v -XGET  -H "Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json" -H "User-Agent: kubectl/v1.27.3 (linux/amd64) kubernetes/25b4e43" 'https://cluster-endpoint:6443/api/v1/pods?limit=500'
I0701 11:07:45.815001  577407 round_trippers.go:495] HTTP Trace: DNS Lookup for cluster-endpoint resolved to [{10.9.37.82 }]
I0701 11:07:45.815204  577407 round_trippers.go:510] HTTP Trace: Dial to tcp:10.9.37.82:6443 succeed
I0701 11:07:45.828484  577407 round_trippers.go:553] GET https://cluster-endpoint:6443/api/v1/pods?limit=500 200 OK in 13 milliseconds
```

从日志可确认是使用 vip 10.9.37.82 发起请求，且请求成功。

### 添加 worker 节点

1. 执行`基础环境配置` 章节各步骤，初始化节点
2. 增加主机映射，如节点 IP 为 10.9.167.63（必须增加 cluster-endpoint 映射）

    ```yaml
    cat >> /etc/hosts<<EOF
    10.9.167.63 node-10-9-167-63
    10.9.37.82 cluster-endpoint
    EOF

    ```

3. 如未保存 join 命令，可在 master 节点重新生成

    ```yaml
    $> kubeadm token create --print-join-command
    kubeadm join cluster-endpoint:6443 --token zs3ugs.6g3rygcf49ctxizq --discovery-token-ca-cert-hash sha256:9a851b28d96879f28027fa670e542dfa55a0c2564b7c6f8ff5c6473ef495b0f8
    ```

4. 执行 `kubectl join` 命令添加节点

    ```yaml
    $> kubeadm join cluster-endpoint:6443 --token zs3ugs.6g3rygcf49ctxizq --discovery-token-ca-cert-hash sha256:9a851b28d96879f28027fa670e542dfa55a0c2564b7c6f8ff5c6473ef495b0f8
    [preflight] Running pre-flight checks
    [WARNING FileExisting-tc]: tc not found in system path
    [preflight] Reading configuration from the cluster...
    [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Starting the kubelet
    [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

    This node has joined the cluster:
    * Certificate signing request was sent to apiserver and a response was received.
    * The Kubelet was informed of the new secure connection details.

    Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
    ```

### 删除 worker 节点

1. master 节点上执行节点删除

    ```bash
    1) 驱逐节点上所有 pod
    $> kubectl drain node-10-9-167-63 --delete-local-data --force --ignore-daemonsets

    2) 确认 pod 全部驱逐
    $> kubectl get pods -A

    3) 删除节点
    $> kubectl delete nodes node-10-9-167-63
    ```

2. 清理节点上脏数据

    ```bash
    $> rm -rf /etc/kubernetes/*
    $> systemctl stop kubelet.service
    ```

## 问题记录

### kubeadm init 报错

```bash
[root@10-9-66-153 ~]# kubeadm init --image-repository registry.aliyuncs.com/google_containers
[init] Using Kubernetes version: v1.27.3
[preflight] Running pre-flight checks
 [WARNING FileExisting-tc]: tc not found in system path
error execution phase preflight: [preflight] Some fatal errors occurred:
 [ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: /etc/kubernetes/manifests/kube-apiserver.yaml already exists
 [ERROR FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml]: /etc/kubernetes/manifests/kube-controller-manager.yaml already exists
 [ERROR FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml]: /etc/kubernetes/manifests/kube-scheduler.yaml already exists
 [ERROR FileAvailable--etc-kubernetes-manifests-etcd.yaml]: /etc/kubernetes/manifests/etcd.yaml already exists
 [ERROR Port-10250]: Port 10250 is in use
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

之前已经运行过多次 `kubeadm init`，导致 yaml 文件冲突，可以增加 `--ignore-preflight-errors=all` 参数忽略错误继续执行；或者先重置（`kubeadm reset`）之后，再执行 `kubeadm init` 。

```bash
kubeadm reset
rm -rf ~/.kube/ /etc/kubernetes/* var/lib/etcd/*
```

### flannel pod 启动异常

```bash
[root@master3 ~]# kubectl get pods -A
NAMESPACE      NAME                              READY   STATUS              RESTARTS      AGE
kube-flannel   kube-flannel-ds-hxtgs             0/1     CrashLoopBackOff    3 (21s ago)   86s
kube-system    coredns-c676cc86f-9wkz6           0/1     ContainerCreating   0             5m6s
kube-system    coredns-c676cc86f-rq4g9           0/1     ContainerCreating   0             5m6s
kube-system    etcd-master3                      1/1     Running             1             5m11s
kube-system    kube-apiserver-master3            1/1     Running             1             5m12s
kube-system    kube-controller-manager-master3   1/1     Running             1             5m11s
kube-system    kube-proxy-wnqcw                  1/1     Running             0             5m6s
kube-system    kube-scheduler-master3            1/1     Running             1             5m11s
```

查看 pod 日志

```bash
[root@master3 ~]# kubectl logs pod -n kube-flannel kube-flannel-ds-hxtgs
Error from server (NotFound): pods "pod" not found
[root@master3 ~]# kubectl logs -n kube-flannel kube-flannel-ds-hxtgs
Defaulted container "kube-flannel" out of: kube-flannel, install-cni-plugin (init), install-cni (init)
I0701 05:34:55.161599       1 main.go:204] CLI flags config: {etcdEndpoints:http://127.0.0.1:4001,http://127.0.0.1:2379 etcdPrefix:/coreos.com/network etcdKeyfile: etcdCertfile: etcdCAFile: etcdUsername: etcdPassword: version:false kubeSubnetMgr:true kubeApiUrl: kubeAnnotationPrefix:flannel.alpha.coreos.com kubeConfigFile: iface:[] ifaceRegex:[] ipMasq:true ifaceCanReach: subnetFile:/run/flannel/subnet.env publicIP: publicIPv6: subnetLeaseRenewMargin:60 healthzIP:0.0.0.0 healthzPort:0 iptablesResyncSeconds:5 iptablesForwardRules:true netConfPath:/etc/kube-flannel/net-conf.json setNodeNetworkUnavailable:true}
W0701 05:34:55.161658       1 client_config.go:617] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0701 05:34:55.259826       1 kube.go:126] Waiting 10m0s for node controller to sync
I0701 05:34:55.259883       1 kube.go:420] Starting kube subnet manager
I0701 05:34:56.260010       1 kube.go:133] Node controller sync successful
I0701 05:34:56.260036       1 main.go:224] Created subnet manager: Kubernetes Subnet Manager - master3
I0701 05:34:56.260045       1 main.go:227] Installing signal handlers
I0701 05:34:56.260138       1 main.go:467] Found network config - Backend type: vxlan
I0701 05:34:56.260153       1 match.go:206] Determining IP address of default interface
I0701 05:34:56.260542       1 match.go:259] Using interface with name eth0 and address 10.9.88.158
I0701 05:34:56.260568       1 match.go:281] Defaulting external address to interface address (10.9.88.158)
I0701 05:34:56.260620       1 vxlan.go:138] VXLAN config: VNI=1 Port=0 GBP=false Learning=false DirectRouting=false
E0701 05:34:56.260846       1 main.go:327] Error registering network: failed to acquire lease: node "master3" pod cidr not assigned
I0701 05:34:56.260914       1 main.go:447] Stopping shutdownHandler...
```

```bash
Error registering network: failed to acquire lease: node "master3" pod cidr not assigned
```

kubeadm init 时需要指定参数 `--pod-network-cidr=10.244.0.0/16`

