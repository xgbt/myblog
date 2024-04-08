---
title: "Kubernetes 1.28 集群安装步骤"
date: 2023-10-01T16:29:53+08:00
draft: false
tags: [K8S]
categories: [实践笔记]
---

**系统配置如下：**

```
OS: AlmaLinux 9.2 (Turquoise Kodkod) x86_64 
Host: VMware7,1 None 
Kernel: 5.14.0-284.30.1.el9_2.x86_64 
Terminal: /dev/pts/0 
CPU: Intel Xeon Gold 6326 (16) @ 2.893GHz 
GPU: 00:0f.0 VMware SVGA II Adapter 
Memory: 3093MiB / 31828MiB
```

- 网络插件使用 Calico
- CRI 使用 Containerd
- kube-proxy 模式为 ipvs


# 1 所有节点配置

## 1.1 基础配置

### 1.1.1 配置镜像源

```sh
# 配置镜像源
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
    -e 's|^#\s*baseurl=https://repo.almalinux.org/almalinux|baseurl=https://mirrors.zju.edu.cn/almalinux|g' \
    -i.bak \
    /etc/yum.repos.d/almalinux-*.repo

# 安装必要软件
yum makecache
yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git -y
```


### 1.1.2 配置 hosts

主节点
```sh
hostnamectl set-hostname zqf-master1 --static
```

从节点
```sh
hostnamectl set-hostname zqf-slave1 --static
```

编辑 `/etc/hosts` 文件

```sh
vim /etc/hosts
```

添加
```
10.101.5.220 zqf-master1
10.101.5.219 zqf-slave1
```

**或者**使用 echo

```
echo "192.168.252.79 master1" >> /etc/hosts
```

### 1.1.3 关闭防火墙

```sh
systemctl stop firewalld && systemctl disable firewalld
systemctl disable --now dnsmasq
```

### 1.1.4 禁用 selinux 

```sh
# 修改配置文件，使 SELINUX=disable
vim /etc/selinux/config

# 或者直接执行下列命令
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/sysconfig/selinux
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config

# 查看系统 SELinux 运行状态
sestatus
```

### 1.1.5 关闭 swap

Swap 是交换分区，如果机器内存不够，会使用swap分区，但是swap分区性能较低，k8s设计的时候为了能提升性能，默认不允许使用交换分区。

kubeadm 初始化的时候会检测 swap 状态，未关闭 swap 会导致初始化失败。


```sh
swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab

# 若 swap 为 0， 说明 swap 关闭成功
free -mh
```



### 1.1.6 配置系统资源限制

```sh
vim /etc/security/limits.conf

# 添加以下配置，包括最大文件描述符数量、最大进程数量和内存锁定限制
* soft nofile 65536
* hard nofile 131072
* soft nproc 65535
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited
```


## 1.2 安装 ipvs

K8S 集群将使用 ipvs 模式，因此这里事先安装 ipvs 相关组件。

```sh
# 安装 ipvs 相关软件包
yum install ipvsadm ipset sysstat conntrack libseccomp

# 载入模块
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack

# 创建ipvs.conf，设置内核模块的自动载入。
cat <<EOF > /etc/modules-load.d/ipvs.conf 
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip
EOF
```

```sh
systemctl enable --now systemd-modules-load.service
```


## 1.3 配置 K8S 内核参数

```sh
cat <<EOF >  /etc/sysctl.d/k8s.conf 
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384
vm.swappiness=0
EOF
```

**重启服务器**

```sh
reboot
```

## 1.4 安装 CRI
### 1.4.1 Docker

docker 与 containerd 并不冲突。

```sh
# 安装 Docker
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加 Docker 源
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 启动并验证 docker 成功安装
systemctl start docker
docker --version
systemctl enable docker
```


### 1.4.2 Contained

**配置 containerd 所需模块**

```sh
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```

```sh
modprobe -- overlay
modprobe -- br_netfilter
```

**配置 containerd 所需内核**

```sh
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

```sh
sysctl --system
```

**生成并编辑 containerd 配置文件**

```sh
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

```sh
vim /etc/containerd/config.toml
```

1. 将 `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]` 条目下的 `SystemdCgroup = false` 修改为 `SystemdCgroup = true`
2. 修改 `sandbox_imag` 地址为 `registry.aliyuncs.com/google_containers/pause:3.9`
3. node 节点配置仓库地址

```
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."10.101.7.108".tls]
                insecure_skip_verify = true
        [plugins."io.containerd.grpc.v1.cri".registry.configs."szharbor.hithium.cn".tls]
                insecure_skip_verify = true
        [plugins."io.containerd.grpc.v1.cri".registry.configs."10.101.7.108".auth]
                username = "zhuangqf"
                password = "@1Xiaoshagua"
        [plugins."io.containerd.grpc.v1.cri".registry.configs."szharbor.hithium.cn".auth]
                username = "zhuangqf"
                password = "@1Xiaoshagua"
      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
                endpoint = ["registry.cn-hangzhou.aliyuncs.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."10.101.7.108:80"]
                endpoint = ["http://10.101.7.108"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."szharbor.hithium.cn"]
                endpoint = ["http://szharbor.hithium.cn/"]
```

**刷新配置**

```sh
systemctl daemon-reload
systemctl enable --now containerd
```

**配置 crictl 客户端连接的运行时位置**

```sh
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

## 1.5 安装 K8S 相关组件 

K8S 所有节点均需要安装 kubectl kubelet kubeadm

```sh
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

```sh
yum -y install kubectl kubelet kubeadm
systemctl enable kubelet.service
```


# 2 主节点配置

## 2.1 基于 kubeadm 安装 K8S 集群

### 2.1.1 修改 kubeadm 配置文件

```sh
kubeadm config print init-defaults > kubeadm-init.yml
```

需要修改的部分包括：
- `localAPIEndpoint.advertiseAddress`：主节点 IP 地址
- `nodeRegistration.name`：主节点 hostname
- `imageRepository`：镜像仓库地址
- `networking.podSubnet`: pod 子网范围

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
  - groups:
      - system:bootstrappers:kubeadm:default-node-token
    token: abcdef.0123456789abcdef
    ttl: 24h0m0s
    usages:
      - signing
      - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.101.5.220  # 主节点 IP
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: zqf-master1             # 主节点 hostname
  taints:
    - effect: NoSchedule
      key: node-role.kubernetes.io/control-plane
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers # 镜像地址
kind: ClusterConfiguration
kubernetesVersion: v1.28.2
networking:
  dnsDomain: cluster.local
  podSubnet: 192.168.0.0/16   # pod 子网 IP 范围
  serviceSubnet: 10.96.0.0/12
scheduler: {}

# 下面是补充
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd

```

### 2.1.2 提前拉取所需镜像

```sh
kubeadm config images pull --config kubeadm-init.yml
```

> **错误：** CRI v1 image API is not implemented for endpoint
> **解决方法：**
> `vim /etc/containerd/config.toml `
> 注释掉 `disabled_plugins = ["cri"]` ，以启用 Kubernetes CRI


### 2.1.3 安装 K8S 集群

```sh
kubeadm init --config=kubeadm-init.yml  --upload-certs
```

其中， `--upload-certs` 会自动将证书从主控制平面节点复制到将要加入的控制平面节点上。

> **错误：** ERROR CRI: container runtime is not running:
> **解决方法：**
> `rm -rf /etc/containerd/config.toml`
> `systemctl restart containerd`

### 2.1.4 配置 kubectl

  ```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
# 将以下命令加到.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
  ```

其中，admin.conf 是连接 Kubernetes 的认证文件，通过此文件才能连接到 kubernetes，kubectl 也需要这个文件；在 Linux 中，使用 KUBECONFIG 环境变量知道认证文件的所在。

Linux 中每个用户的环境变量是不同的，如果切换了用户，则也需要设置 KUBECONFIG 环境变量；如果要在别的节点上连接集群，则可以把这个文件复制过去。


## 2.2 安装 Calico

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/tigera-operator.yaml

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/custom-resources.yaml
```

> **问题：**
> 安装 Calico 之后，会出现 calico-node 已经部署完成，但是 node 状态依旧为 not-ready 的情况，可能与 core-dns 的重启策略有关。
> **解决方法：**
> 依次 reboot 重启主节点和从节点

## 2.3 配置 kube-proxy 为 ipvs


```sh
# 将 mode 改为 'ipvs'
kubectl edit cm kube-proxy -n kube-system

# 更新 kube-proxy 对应 pod
kubectl -n kube-system rollout restart daemonset kube-proxy

# 检查 kube-proxy 模式
curl 127.0.0.1:10249/proxyMode
```


## 2.4 配置 kubectl 命令补全

```bash
yum install -y bash-completion 
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

# 3 清除 kubeadm 环境


```sh
kubeadm reset cleanup-node
kubeadm reset
rm -rf /etc/cni/net.d
ipvsadm --clear
rm -rf $HOME/.kube/config
```




# 参考资料

1. [Kubernetes 1.25 集群保姆级安装教程](https://zhpengfei.com/kubernetes-cluster-installation-guide-with-containerd-and-calico/#aioseo-1-9-k8s)
2. [国内自建公网 Kubernetes 指南](https://val-istar-guo.com/articles/4.%E5%9B%BD%E5%86%85%E8%87%AA%E5%BB%BAKubernetes%E6%8C%87%E5%8D%97.md#%E5%AE%89%E8%A3%85runc)