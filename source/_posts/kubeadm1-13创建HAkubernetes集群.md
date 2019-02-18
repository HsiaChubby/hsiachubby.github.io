---
title: kubeadm1.13创建HAkubernetes集群
date: 2019-02-18 10:13:14
tags: kubernetes
categories: 大数据
---
[TOC]
## 一、环境准备
### 1.1 硬件设备环境
采用5台腾讯云的CVM作为kubernetes的部署环境，具体信息如下：

| 主机名 | IP | 配置 | 备注 |
| --- | --- | --- | --- |
| （Old）VM_0_1_centos；（New）tf-k8s-m1 | 10.0.0.1 | 4c 8g | k8s的master，同时也是etcd节点 |
| （Old）VM_0_2_centos；（New）tf-k8s-m2 | 10.0.0.2 | 4c 8g | k8s的master，同时也是etcd节点 |
| （Old）VM_0_3_centos；（New）tf-k8s-m3 | 10.0.0.3 | 4c 8g | k8s的master，同时也是etcd节点 |
| （Old）VM_0_4_centos；（New）tf-k8s-n1 | 10.0.0.4 | 4c 8g | 工作节点 node，容器编排最终 pod 工作节点 |
| （Old）VM_0_5_centos；（New）tf-k8s-n2 | 10.0.0.5 | 4c 8g | 工作节点 node，容器编排最终 pod 工作节点 |

### 1.2 软件环境

| 环境 | 简介 |
| --- | --- |
| 操作系统 | CentOS7 |
| kubeadm | 1.13.3 |
| kubernetes | 1.13.3 |
| Docker | docker-ce 18.06.2 |

### 1.3 相关系统设置
在正式安装之前，需要在每台机器上对以下配置进行修改：
* 关闭防火墙，selinux
* 关闭系统的swap功能
* 关闭Linux swap空间的swappiness
* 配置L2网桥在转发包时会被iptables的FORWARD规则所过滤，该配置被CNI插件需要，更多信息请参考[Network Plugin Requirements](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#network-plugin-requirements)
* 升级内核到最新（centos7 默认的内核是3.10.0-862.el7.x86_64 ,可以使用命令‘uname -a’进行查看），原因见[请问下为什么要用4.18版本内核](https://github.com/Lentil1016/kubeadm-ha/issues/19)
* 开启IPVS
* 修改主机名（如果主机名中含有一些特殊字符，则需要调整主机名，不然在后续操作中会出现错误）
具体的配置修改执行脚本如下：

```
# ---------- 关闭防火墙和selinux -----------
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

# ---------- 关闭交换分区 -----------
swapoff -a
yes | cp /etc/fstab /etc/fstab_bak
cat /etc/fstab_bak |grep -v swap > /etc/fstab

# ---------- 设置网桥包经IPTables，core文件生成路径 -----------
echo """
vm.swappiness = 0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
""" > /etc/sysctl.conf
modprobe br_netfilter
sysctl -p

# ---------- 同步时间 -----------
yum install -y ntpdate
ntpdate -u ntp.api.bz

# ---------- 升级内核 -----------
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm ;yum --enablerepo=elrepo-kernel install kernel-ml-devel kernel-ml -y
# 查看启动配置里是否有最新的内核
cat /boot/grub2/grub.cfg | grep menuentry
# 修改默认启动项
grub2-set-default 0
# 检查默认内核版本是否大于4.14，否则请调整默认启动参数
grub2-editenv list
#重启以更换内核
reboot
#查看内核信息
uname -a

# ---------- 确认内核版本后，开启IPVS -----------
uname -a
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_fo ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack"
for kernel_module in \${ipvs_modules}; do
 /sbin/modinfo -F filename \${kernel_module} > /dev/null 2>&1
 if [ $? -eq 0 ]; then
 /sbin/modprobe \${kernel_module}
 fi
done
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs

# ---------- 修改主机名 -----------
# 这里以VM_0_17_centos主机为例，其他的主机分别修改成相应的主机名
hostnamectl set-hostname tf-k8s-m1 
```
### 1.3 配置集群内各个机器之间的免密码登录
#### 1.3.1 配置hosts 
为了便于后续的操作，我们需要给每一台设备配置下hosts域名信息，具体如下：

```
# vi /etc/hosts
10.0.0.1 tf-k8s-m1 api.tf-k8s.xiangwushuo.com
10.0.0.2 tf-k8s-m2
10.0.0.3 tf-k8s-m3
10.0.0.4 tf-k8s-n1
10.0.0.5 tf-k8s-n2
10.0.0.1 dashboard.tf-k8s.xiangwushuo.com
```
#### 1.3.2 新建用户
```
# useradd kube
# visudo
%wheel  ALL=(ALL)       ALL
kube    ALL=(ALL)       NOPASSWD:ALL
```
备注：visudo命令是用来给kube用户添加sudo密码
#### 1.3.3 设置免密登录
1. 各个设备的root用户&kube用户（不同用户配置不同的）都生成各自的免密登录的ssh的私钥与公钥

```
## 为root用户生成ssh的私钥与公钥
ssh-keygen
```
在/root目录下，会生成一个.ssh目录，.ssh目录下会生成以下三个文件：

```
-rw------- 1 root root 2398 Feb 13 15:18 authorized_keys
-rw------- 1 root root 1679 Feb 13 14:47 id_rsa
-rw-r--r-- 1 root root  401 Feb 13 14:47 id_rsa.pub
```
authorized_keys文件存储了本设备认证授权的其他设备的公钥信息；id_rsa存储了本设备的私钥信息；id_rsa.pub存储了本设备的公钥信息。

```
## 为kube用户生成ssh的私钥与公钥
su kube
ssh-keygen
```
在/home/kube目录下，会生成一个.ssh目录，并包含相关文件。
2. 各个设备上都创建好各自的ssh免密登录公钥与私钥后，需要将各自的公钥copy至其他的设备上，并将公钥信息添加到各个设备的authorized_keys文件中。
备注：也需要将各个节点自己的公钥copy至自己的authorized_keys文件中，这样自己才可以ssh自己。

```
## 将每一台节点上的公钥都同步到相应的目录下
# ll
-rw-r--r-- 1 root root  401 Feb 13 15:15 tf-k8s-m1-id_rsa.pub
-rw-r--r-- 1 root root  401 Feb 13 15:15 tf-k8s-m2-id_rsa.pub
-rw-r--r-- 1 root root  401 Feb 13 15:15 tf-k8s-m3-id_rsa.pub
-rw-r--r-- 1 root root  401 Feb 13 15:15 tf-k8s-n1-id_rsa.pub
-rw-r--r-- 1 root root  401 Feb 13 15:15 tf-k8s-n2-id_rsa.pub
## 将每台节点的公钥追加至authorized_keys文件中
cat tf-k8s-m1-id_rsa.pub > authorized_keys
cat tf-k8s-m2-id_rsa.pub > authorized_keys
cat tf-k8s-m3-id_rsa.pub > authorized_keys
cat tf-k8s-n1-id_rsa.pub > authorized_keys
cat tf-k8s-n2-id_rsa.pub > authorized_keys
```
测试是否能够正常使用ssh免密登录

```
ssh root@tf-k8s-m1
ssh root@tf-k8s-m2
ssh root@tf-k8s-m3
ssh root@tf-k8s-n1
ssh root@tf-k8s-n2
```
**提示**：如果其他机器上的 root 下的 /root/.ssh/authorized_keys 不存在，可以手动创建。要注意的是：authorized_keys 的权限需要是 600。

```
## 如果 authorized_keys 的权限不是 600，执行修改权限的命令。
chmod 600 authorized_keys
```

## 二、安装步骤
以下操作，可以都切换至kube用户下进行操作。
### 2.1 安装docker
由于kubeadm的ha模式对docker的版本是有一定的要求的，因此，本教程中安装官方推荐的docker版本。

```
# 安装依赖包
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加Docker软件包源
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

#关闭测试版本list（只显示稳定版）
sudo yum-config-manager --enable docker-ce-edge
sudo yum-config-manager --enable docker-ce-test

# 更新yum包索引
yum makecache fast

#NO.1 指定版本安装
yum list docker-ce --showduplicates|sort -r  
yum install docker-ce-18.06.2.ce -y
```
为了方便操作，我们在tf-k8s-m1节点上，创建一个批量部署docker的脚本。

```
## 创建install.docker.sh

#!/bin/sh

vhosts="tf-k8s-m1 tf-k8s-m2 tf-k8s-m3 tf-k8s-n1 tf-k8s-n2"

for h in $vhosts
do
    echo "Install docker for $h"
    ssh kube@$h "sudo yum install docker-ce-18.06.2.ce -y && sudo systemctl enable docker && systemctl start docker"
done 
```
执行install.docker.sh脚本

```
chmod a+x install.docker.sh
sh ./install.docker.sh
```

### 2.2 安装kubernetes yum源和kubelet、kubeadm、kubectl组件
#### 2.2.1 所有机器上配置 kubernetes.repo yum 源
详细的安装脚本如下：

```
## 创建脚本：install.k8s.repo.sh

#!/bin/sh

vhost="tf-k8s-m1 tf-k8s-m2 tf-k8s-m3 tf-k8s-n1 tf-k8s-n2"

## 设置为阿里云 kubernetes 仓库
cat <<EOF > kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

mvCmd="sudo cp ~/kubernetes.repo /etc/yum.repos.d/"
for h in $vhost
do
  echo "Setup kubernetes repository for $h"
  scp ./kubernetes.repo kube@$h:~
  ssh kube@$h $mvCmd
done

```
执行install.k8s.repo.sh脚本

```
chmod a+x install.k8s.repo.sh
sh ./install.k8s.repo.sh
```

#### 2.2.2 所有机器上安装 kubelet、kubeadm、kubectl组件
详细安装脚本如下：

```
## 创建脚本：install.k8s.basic.sh

#!/bin/sh

vhost="tf-k8s-m1 tf-k8s-m2 tf-k8s-m3 tf-k8s-n1 tf-k8s-n2"

## 安装 kubelet kubeadm kubectl
installCmd="sudo yum install -y kubelet kubeadm kubectl && sudo systemctl enable kubelet"
for h in $vhost
do
  echo "Install kubelet kubeadm kubectl for : $h"
  ssh kube@$h $installCmd
done
```
执行install.k8s.baisc.sh脚本

```
chmod a+x install.k8s.basic.sh
sh ./install.k8s.basic.sh
```
### 2.3 初始化kubeadm配置文件
创建三台master机器tf-k8s-m1,tf-k8s-m2,tf-k8s-m3的kubeadm配置文件，其中主要是配置生成证书的域配置、etcd集群配置。

```
## 创建脚本：init.kubeadm.config.sh

#!/bin/sh

## 1. 配置参数 
## vhost 主机名和 vhostIP IP 一一对应
vhost=(tf-k8s-m1 tf-k8s-m2 tf-k8s-m3)
vhostIP=(10.0.0.1 10.0.0.2 10.0.0.3)

domain=api.tf-k8s.xiangwushuo.com

## etcd 初始化 m01 m02 m03 集群配置
etcdInitCluster=(
tf-k8s-m1=https://10.0.0.1:2380
tf-k8s-m1=https://10.0.0.1:2380,tf-k8s-m2=https://10.0.0.2:2380
tf-k8s-m1=https://10.0.0.1:2380,tf-k8s-m2=https://10.0.0.2:2380,tf-k8s-m3=https://10.0.0.3:2380
)

## etcd 初始化时，m01 m02 m03 分别的初始化集群状态
initClusterStatus=(
new
existing
existing
)


## 2.遍历 master 主机名和对应 IP
## 生成对应的 kubeadmn 配置文件 
for i in `seq 0 $((${#vhost[*]}-1))`
do

h=${vhost[${i}]} 
ip=${vhostIP[${i}]}

echo "--> $h - $ip"
  
## 生成 kubeadm 配置模板
cat <<EOF > kubeadm-config.$h.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: $ip
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.13.3

# 指定阿里云镜像仓库
imageRepository: registry.aliyuncs.com/google_containers

# apiServerCertSANs 填所有的 masterip、lbip、其它可能需要通过它访问 apiserver 的地址、域名或主机名等，
# 如阿里fip，证书中会允许这些ip
# 这里填一个自定义的域名
apiServer:
  certSANs:
  - "$domain"
controlPlaneEndpoint: "$domain:6443"

## Etcd 配置
etcd:
  local:
    extraArgs:
      listen-client-urls: "https://127.0.0.1:2379,https://$ip:2379"
      advertise-client-urls: "https://$ip:2379"
      listen-peer-urls: "https://$ip:2380"
      initial-advertise-peer-urls: "https://$ip:2380"
      initial-cluster: "${etcdInitCluster[${i}]}"
      initial-cluster-state: ${initClusterStatus[${i}]}
    serverCertSANs:
      - $h
      - $ip
    peerCertSANs:
      - $h
      - $ip
networking:
  podSubnet: "10.244.0.0/16"

EOF

echo "kubeadm-config.$h.yaml created ... ok"

## 3. 分发到其他 master 机器 
scp kubeadm-config.$h.yaml kube@$h:~
echo "scp kubeadm-config.$h.yaml ... ok"

done
```
执行init.kubeadm.config.sh脚本

```
chmod a+x init.kubeadm.config.sh
sh ./init.kubeadm.config.sh
```
执行成功之后，可以在tf-k8s-m1, tf-k8s-m2, tf-k8s-m3的 kube 用户的 home 目录（/home/kube）能看到对应的 kubeadm-config.tf-k8s-m1*.yaml 配置文件。 这个配置文件主要是用于后续初始化集群其他 master 的证书、 etcd 配置、kubelet 配置、kube-apiserver配置、kube-controller-manager 配置等。
各个master节点上对应的kubeadm配置文件：

```
cvm tf-k8s-m1：kubeadm-config.tf-k8s-m1.yaml
cvm tf-k8s-m2：kubeadm-config.tf-k8s-m2.yaml
cvm tf-k8s-m3：kubeadm-config.tf-k8s-m3.yaml
```

### 2.4 安装master镜像和执行kubeadm初始化
#### 2.4.1 拉取镜像到本地
因为 k8s.gcr.io 国内无法访问，我们可以选择通过阿里云的镜像仓库（kubeadm-config.tf-k8s-m1*.yaml 配置文件中已经指定使用阿里云镜像仓库  registry.aliyuncs.com/google_containers），将所需的镜像 pull 到本地。
我们可以通过以下命令，来查看是否已经成功指定了阿里云的镜像仓库,在tf-k8s-m1机器上，通过`kubeadm config images list`命令来查看，结果如下:

```
[kube@tf-k8s-m1 ~]$ kubeadm config images list --config kubeadm-config.tf-k8s-m1.yaml
registry.aliyuncs.com/google_containers/kube-apiserver:v1.13.3
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.13.3
registry.aliyuncs.com/google_containers/kube-scheduler:v1.13.3
registry.aliyuncs.com/google_containers/kube-proxy:v1.13.3
registry.aliyuncs.com/google_containers/pause:3.1
registry.aliyuncs.com/google_containers/etcd:3.2.24
registry.aliyuncs.com/google_containers/coredns:1.2.6

```
接下来，分别在tf-k8s-m1、tf-k8s-m2、tf-k8s-m3机器上，拉取相关镜像

```
[kube@tf-k8s-m1 ~]$ sudo kubeadm config images pull --config kubeadm-config.tf-k8s-m1.yaml
[kube@tf-k8s-m2 ~]$ sudo kubeadm config images pull --config kubeadm-config.tf-k8s-m2.yaml
[kube@tf-k8s-m3 ~]$ sudo kubeadm config images pull --config kubeadm-config.tf-k8s-m3.yaml
```
执行成功后，应该能够看到本地已经拉取的镜像

```
[kube@tf-k8s-m1 ~]$ sudo docker images
REPOSITORY                                                                     TAG                 IMAGE ID            CREATED             SIZE
registry.aliyuncs.com/google_containers/kube-apiserver                         v1.13.3             fe242e556a99        2 weeks ago         181MB
registry.aliyuncs.com/google_containers/kube-proxy                             v1.13.3             98db19758ad4        2 weeks ago         80.3MB
registry.aliyuncs.com/google_containers/kube-controller-manager                v1.13.3             0482f6400933        2 weeks ago         146MB
registry.aliyuncs.com/google_containers/kube-scheduler                         v1.13.3             3a6f709e97a0        2 weeks ago         79.6MB
quay.io/coreos/flannel                                                         v0.11.0-amd64       ff281650a721        2 weeks ago         52.6MB
registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller   0.21.0              01bd760f276c        2 months ago        568MB
registry.aliyuncs.com/google_containers/coredns                                1.2.6               f59dcacceff4        3 months ago        40MB
registry.aliyuncs.com/google_containers/etcd                                   3.2.24              3cab8e1b9802        5 months ago        220MB
registry.aliyuncs.com/google_containers/pause                                  3.1                 da86e6ba6ca1        14 months ago       742kB
```

#### 2.4.2 安装master tf-k8s-m1
我们目标是要搭建一个高可用的 master 集群，所以需要在三台 master tf-k8s-m1 tf-k8s-m2 tf-k8s-m3机器上分别通过 kubeadm 进行初始化。
由于 tf-k8s-m2 和 tf-k8s-m3 的初始化需要依赖 tf-k8s-m1 初始化成功后所生成的证书文件，所以这里需要先在 m01 初始化。

```
[kube@tf-k8s-m1 ~]$  sudo kubeadm init --config kubeadm-config.tf-k8s-m1.yaml 
```
初始化成功后，会看到如下日志：
**备注：如果初始化失败，则可以通过`kubeadm reset --force`命令重置之前kubeadm init命令的执行结果，恢复一个干净的环境**

```
[init] Using Kubernetes version: v1.13.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [m01 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local api.k8s.hiko.im api.k8s.hiko.im] and IPs [10.96.0.1 10.0.2.15]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [m01 localhost m01] and IPs [10.0.2.15 127.0.0.1 ::1 192.168.33.10]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [m01 localhost m01] and IPs [10.0.2.15 127.0.0.1 ::1 192.168.33.10]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 19.009523 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "m01" as an annotation
[mark-control-plane] Marking the node m01 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node m01 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: a1t7c1.mzltpc72dc3wzj9y
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join api.k8s.hiko.im:6443 --token a1t7c1.mzltpc72dc3wzj9y --discovery-token-ca-cert-hash sha256:05f44b111174613055975f012fc11fe09bdcd746bd7b3c8d99060c52619f8738

```
至此，就完成了第一台master的初始化工作。

#### 2.4.3 kube用户配置
为了让tf-k8s-m1的 kube 用户能通过 kubectl 管理集群，接着我们需要给tf-k8s-m1 的 kube 用户配置管理集群的配置。在tf-k8s-m1机器上创建config.using.cluster.sh脚本，具体如下：

```
## 创建脚本：config.using.cluster.sh

#!/bin/sh

# 为 kube 用户配置
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
执行config.using.cluster.sh脚本

```
chmod a+x config.using.cluster.sh
sh ./config.using.cluster.sh
```
验证结果，通过`kubectl`命令查看集群状态，结果如下：

```
[kube@tf-k8s-m1 ~]$ kubectl cluster-info
Kubernetes master is running at https://api.tf-k8s.xiangwushuo.com:6443
KubeDNS is running at https://api.tf-k8s.xiangwushuo.com:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
查看集群所有的pods信息，结果如下：

```
[kube@tf-k8s-m1 ~]$ kubectl get pods --all-namespaces

NAMESPACE     NAME                          READY   STATUS    RESTARTS   AGE
kube-system   coredns-78d4cf999f-cw79l      0/1     Pending   0          47m
kube-system   coredns-78d4cf999f-w8j47      0/1     Pending   0          47m
kube-system   etcd-m01                      1/1     Running   0          47m
kube-system   kube-apiserver-m01            1/1     Running   0          46m
kube-system   kube-controller-manager-m01   1/1     Running   0          46m
kube-system   kube-proxy-5954k              1/1     Running   0          47m
kube-system   kube-scheduler-m01            1/1     Running   0          47m
```
其中，由于未安装相关的网络组件，eg:flannel,所有coredn还是显示为pending，暂时没有影响。

#### 2.4.4 安装CNI插件flannel
**备注：所有的节点都需要安装**
具体的安装脚本如下：

```
## 拉取镜像
sudo docker pull quay.io/coreos/flannel:v0.11.0-amd64
## 部署
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
安装成功之后，通过 `kubectl get pods --all-namespaces`，看到所有 Pod 都正常了.

### 2.5 安装剩余的master
#### 2.5.1 同步tf-k8s-m1的ca证书
首先，将 tf-k8s-m1 中的 ca 证书，scp 到其他 master 机器（tf-k8s-m2 tf-k8s-m3）。
为了方便，这里也是通过脚本来执行，具体如下：
注意：需要确认 tf-k8s-m1 上的 root 账号可以免密登录到 tf-k8s-m2 和 tf-k8s-m3 的 root 账号。

```
## 创建脚本：sync.master.ca.sh

#!/bin/sh

vhost="tf-k8s-m2 tf-k8s-m3"
usr=root

who=`whoami`
if [[ "$who" != "$usr" ]];then
  echo "请使用 root 用户执行或者 sudo ./sync.master.ca.sh"
  exit 1
fi

echo $who

# 需要从 m01 拷贝的 ca 文件
caFiles=(
/etc/kubernetes/pki/ca.crt
/etc/kubernetes/pki/ca.key
/etc/kubernetes/pki/sa.key
/etc/kubernetes/pki/sa.pub
/etc/kubernetes/pki/front-proxy-ca.crt
/etc/kubernetes/pki/front-proxy-ca.key
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/etcd/ca.key
/etc/kubernetes/admin.conf
)

pkiDir=/etc/kubernetes/pki/etcd
for h in $vhost 
do

  ssh ${usr}@$h "mkdir -p $pkiDir"
  
  echo "Dirs for ca scp created, start to scp..."

  # scp 文件到目标机
  for f in ${caFiles[@]}
  do 
    echo "scp $f ${usr}@$h:$f"
    scp $f ${usr}@$h:$f
  done

  echo "Ca files transfered for $h ... ok"
done
```
执行脚本，将 tf-k8s-m1 相关的 ca 文件传到tf-k8s-m2 和 tf-k8s-m3：

```
chmod +x sync.master.ca.sh

sudo ./syncaster.ca.sh
```

#### 2.5.2 安装master tf-k8s-m2
总共分为四个步骤，分别是:总1. 共分为四个步骤，分别是:
* 配置证书、初始化 kubelet 配置和启动 kubelet

```
[kube@tf-k8s-m2 ~]$ sudo kubeadm init phase certs all --config kubeadm-config.tf-k8s-m2.yaml
[kube@tf-k8s-m2 ~]$ sudo kubeadm init phase etcd local --config kubeadm-config.tf-k8s-m2.yaml
[kube@tf-k8s-m2 ~]$ sudo kubeadm init phase kubeconfig kubelet --config kubeadm-config.tf-k8s-m2.yaml
[kube@tf-k8s-m2 ~]$ sudo kubeadm init phase kubelet-start --config kubeadm-config.tf-k8s-m2.yaml
```
* 将etcd加入集群

```
[kube@tf-k8s-m2 root]$ kubectl exec -n kube-system etcd-tf-k8s-m1 -- etcdctl --ca-file /etc/kubernetes/pki/etcd/ca.crt --cert-file /etc/kubernetes/pki/etcd/peer.crt --key-file /etc/kubernetes/pki/etcd/peer.key --endpoints=https://10.0.0.1:2379 member add tf-k8s-m2 https://10.0.0.2:2380
```

启动kube-apiserver、kube-controller-manager、kube-scheduler

```
[kube@tf-k8s-m2 ~]$ sudo kubeadm init phase kubeconfig all --config kubeadm-config.m02.yaml
[kube@tf-k8s-m2 ~]$ sudo kubeadm init phase control-plane all --config kubeadm-config.m02.yaml
```
将节点标记为master节点

```
[kube@tf-k8s-m2 ~]$ sudo kubeadm init phase mark-control-plane --config kubeadm-config.m02.yaml
```

#### 2.5.3 安装master tf-k8s-m3
安装过程和安装master tf-k8s-m2是一样的，区别在于使用的kubeadm配置文件为kubeadm-config.tf-k8s-m3.yaml以及etcd加入成员时指定的实例地址不一样。
完整的流程如下:

```
# 1.  配置证书、初始化 kubelet 配置和启动 kubelet
sudo kubeadm init phase certs all --config kubeadm-config.tf-k8s-m3.yaml
sudo kubeadm init phase etcd local --config kubeadm-config.tf-k8s-m3.yaml
sudo kubeadm init phase kubeconfig kubelet --config kubeadm-config.tf-k8s-m3.yaml
sudo kubeadm init phase kubelet-start --config kubeadm-config.tf-k8s-m3.yaml

# 2. 将 etcd 加入集群
kubectl exec -n kube-system etcd-tf-k8s-m1 -- etcdctl --ca-file /etc/kubernetes/pki/etcd/ca.crt --cert-file /etc/kubernetes/pki/etcd/peer.crt --key-file /etc/kubernetes/pki/etcd/peer.key --endpoints=https://10.0.0.1:2379 member add tf-k8s-m3 https://10.0.0.3:2380

# 3. 启动 kube-apiserver、kube-controller-manager、kube-scheduler
sudo kubeadm init phase kubeconfig all --config kubeadm-config.tf-k8s-m3.yaml
sudo kubeadm init phase control-plane all --config kubeadm-config.tf-k8s-m3.yaml

# 4. 将节点标记为 master 节点
sudo kubeadm init phase mark-control-plane --config kubeadm-config.tf-k8s-m3.yaml
```
#### 2.5.4 验证三个master节点
至此，三个 master 节点安装完成，通过 kubectl get pods --all-namespaces 查看当前集群所有 Pod。

```
[kube@tf-k8s-m2 ~]$ kubectl  get pods --all-namespaces
NAMESPACE     NAME                          READY   STATUS    RESTARTS   AGE
kube-system   coredns-78d4cf999f-j8zsr      1/1     Running   0          170m
kube-system   coredns-78d4cf999f-lw5qx      1/1     Running   0          171m
kube-system   etcd-m01                      1/1     Running   8          5h11m
kube-system   etcd-m02                      1/1     Running   12         97m
kube-system   etcd-m03                      1/1     Running   0          91m
kube-system   kube-apiserver-m01            1/1     Running   9          5h11m
kube-system   kube-apiserver-m02            1/1     Running   0          95m
kube-system   kube-apiserver-m03            1/1     Running   0          91m
kube-system   kube-controller-manager-m01   1/1     Running   4          5h11m
kube-system   kube-controller-manager-m02   1/1     Running   0          95m
kube-system   kube-controller-manager-m03   1/1     Running   0          91m
kube-system   kube-flannel-ds-amd64-7b86z   1/1     Running   0          3h31m
kube-system   kube-flannel-ds-amd64-98qks   1/1     Running   0          91m
kube-system   kube-flannel-ds-amd64-ljcdp   1/1     Running   0          97m
kube-system   kube-proxy-krnjq              1/1     Running   0          5h12m
kube-system   kube-proxy-scb25              1/1     Running   0          91m
kube-system   kube-proxy-xp4rj              1/1     Running   0          97m
kube-system   kube-scheduler-m01            1/1     Running   4          5h11m
kube-system   kube-scheduler-m02            1/1     Running   0          95m
kube-system   kube-scheduler-m03            1/1     Running   0          91m
```
#### 2.5.5 加入工作节点
这步很简单，只需要在工作节点 tf-k8s-n1 和 tf-k8s-n2 上执行加入集群的命令即可。

可以使用上面安装 master tf-k8s-m1 成功后打印的命令 kubeadm join api.tf-k8s.xiangwushuo.com:6443 --token a1t7c1.mzltpc72dc3wzj9y --discovery-token-ca-cert-hash sha256:05f44b111174613055975f012fc11fe09bdcd746bd7b3c8d99060c52619f8738，也可以重新生成 Token。
这里演示如何重新生成 Token 和 证书 hash，在 tf-k8s-m1 上执行以下操作：

```
# 1. 创建 token
[kube@tf-k8s-m1 ~]$ kubeadm token create 

# 控制台打印如：
gz1v4w.sulpuxkqtnyci92f

# 2.  查看我们创建的 k8s 集群的证书 hash
[kube@tf-k8s-m1 ~]$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

# 控制台打印如：
b125cd0c80462353d8fa3e4f5034f1e1a1e3cc9bade32acfb235daa867c60f61
```
然后使用`kubeadm join`,分别在工作节点tf-k8s-n1与tf-k8s-n2上执行，将节点加入
集群，如下：

```
sudo kubeadm join api.tf-k8s.xiangwushuo.com:6443 --token gz1v4w.sulpuxkqtnyci92f --discovery-token-ca-cert-hash sha256:b125cd0c80462353d8fa3e4f5034f1e1a1e3cc9bade32acfb235daa867c60f61
```

在 tf-k8s-m1 上通过 kubectl get nodes 查看，将看到节点已被加进来（节点刚加进来时，状态可能会是 NotReady，稍等一会就回变成 Ready）。

### 2.6 部署高可用CoreDNS
默认安装的 CoreDNS 存在单点问题。在 m01 上通过 kubectl get pods -n kube-system -owide 查看当前集群 CoreDNS Pod 分布（如下）。

从列表中，可以看到 CoreDNS 的两个 Pod 都在 m01 上，存在单点问题。

```
[kube@tf-k8s-m1 ~]$ kubectl get pods -n kube-system -owide
NAME                                    READY   STATUS    RESTARTS   AGE     IP            NODE        NOMINATED NODE   READINESS GATES
coredns-6c67f849c7-h7lcr                1/1     Running   0          4d3h    10.244.3.2    tf-k8s-m1   <none>           <none>
coredns-6c67f849c7-mx9k9                1/1     Running   0          4d3h    10.244.4.2    tf-k8s-m1   <none>           <none>
etcd-tf-k8s-m1                          1/1     Running   1          4d5h    10.0.0.1   tf-k8s-m1   <none>           <none>
etcd-tf-k8s-m2                          1/1     Running   7          4d3h    10.0.0.2   tf-k8s-m2   <none>           <none>
etcd-tf-k8s-m3                          1/1     Running   7          4d3h    10.0.0.3   tf-k8s-m3   <none>           <none>
kube-apiserver-tf-k8s-m1                1/1     Running   0          4d5h    10.0.0.1   tf-k8s-m1   <none>           <none>
kube-apiserver-tf-k8s-m2                1/1     Running   0          4d3h    10.0.0.2   tf-k8s-m2   <none>           <none>
kube-apiserver-tf-k8s-m3                1/1     Running   0          4d3h    10.0.0.3   tf-k8s-m3   <none>           <none>
kube-controller-manager-tf-k8s-m1       1/1     Running   1          4d5h    10.0.0.1   tf-k8s-m1   <none>           <none>
kube-controller-manager-tf-k8s-m2       1/1     Running   0          4d3h    10.0.0.2   tf-k8s-m2   <none>           <none>
kube-controller-manager-tf-k8s-m3       1/1     Running   0          4d3h    10.0.0.3   tf-k8s-m3   <none>           <none>
kube-flannel-ds-amd64-4v6dd             1/1     Running   1          4d3h    10.0.0.5   tf-k8s-n2   <none>           <none>
kube-flannel-ds-amd64-g6sg5             1/1     Running   0          4d3h    10.0.0.3   tf-k8s-m3   <none>           <none>
kube-flannel-ds-amd64-ml4w7             1/1     Running   1          4d3h    10.0.0.4   tf-k8s-n1   <none>           <none>
kube-flannel-ds-amd64-tb27x             1/1     Running   0          4d3h    10.0.0.2   tf-k8s-m2   <none>           <none>
kube-flannel-ds-amd64-x5dqj             1/1     Running   0          4d4h    10.0.0.1   tf-k8s-m1   <none>           <none>
kube-proxy-4wbn7                        1/1     Running   0          4d3h    10.0.0.4   tf-k8s-n1   <none>           <none>
kube-proxy-8dhtz                        1/1     Running   0          4d3h    10.0.0.2   tf-k8s-m2   <none>           <none>
kube-proxy-l8727                        1/1     Running   0          4d5h    10.0.0.1   tf-k8s-m1   <none>           <none>
kube-proxy-tz924                        1/1     Running   0          4d3h    10.0.0.5   tf-k8s-n2   <none>           <none>
kube-proxy-w7tmn                        1/1     Running   0          4d3h    10.0.0.3   tf-k8s-m3   <none>           <none>
kube-scheduler-tf-k8s-m1                1/1     Running   1          4d5h    10.0.0.1   tf-k8s-m1   <none>           <none>
kube-scheduler-tf-k8s-m2                1/1     Running   0          4d3h    10.0.0.2   tf-k8s-m2   <none>           <none>
kube-scheduler-tf-k8s-m3                1/1     Running   0          4d3h    10.0.0.3   tf-k8s-m3   <none>           <none>
kubernetes-dashboard-847f8cb7b8-hmf9m   1/1     Running   0          3d23h   10.244.4.4    tf-k8s-n2   <none>           <none>
metrics-server-8658466f94-pzl6z         1/1     Running   0          4d2h    10.244.3.3    tf-k8s-n1   <none>           <none>
```
首先删除CoreDNS的deploy，然后创建新的CoreDNS-HA.yaml配置文件，如下

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: kube-dns
  name: coredns
  namespace: kube-system
spec:
  #集群规模可自行配置
  replicas: 2
  selector:
    matchLabels:
      k8s-app: kube-dns
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: k8s-app
                  operator: In
                  values:
                  - kube-dns
              topologyKey: kubernetes.io/hostname
      containers:
      - args:
        - -conf
        - /etc/coredns/Corefile
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.2.6
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/coredns
          name: config-volume
          readOnly: true
      dnsPolicy: Default
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: coredns
      serviceAccountName: coredns
      terminationGracePeriodSeconds: 30
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: Corefile
            path: Corefile
          name: coredns
        name: config-volume
```
部署新的CoreDNS 

```
kubectl apply -f CoreDNS-HA.yaml
```

### 2.7 部署监控组件metrics-server

kubernetesv1.11 以后不再支持通过 heaspter 采集监控数据。使用新的监控数据采集组件metrics-server。 metrics-server 比 heaspter 轻量很多，也不做数据的持久化存储，提供实时的监控数据查询。

先将所有文件下载，保存在一个文件夹 metrics-server 里。

修改 metrics-server-deployment.yaml 两处地方，分别是：apiVersion 和 image，最终修改后的 metrics-server-deployment.yaml 如下：

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
# 将extensions/v1beta1修改为apps/v1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: cloudnil/metrics-server-amd64:v0.3.1
        command:
          - /metrics-server
          - --kubelet-insecure-tls
          - --kubelet-preferred-address-types=InternalIP
        imagePullPolicy: Always
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
```
进入刚创建的 metrics-server，执行 kubectl apply -f .  进行部署（注意 -f 后面有个点）,如下：

```
[kube@tf-k8s-m1 metrics-server]$ kubectl apply -f .

clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
serviceaccount/metrics-server created
deployment.apps/metrics-server created
service/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
```
运行`kubectl get pods -n kube-system`，确定metrics-server的pods是否正常running。

### 2.8 部署Nginx-ingress-controller
Nginx-ingress-controller 是 kubernetes 官方提供的集成了 Ingress-controller 和 Nginx 的一个 docker 镜像。

本次部署中，将 Nginx-ingress 部署到 tf-k8s-m1、tf-k8s-m2、tf-k8s-m3上，监听宿主机的 80 端口。

创建 nginx-ingress.yaml 文件，内容如下：

```
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
    resources:
      - ingresses/status
    verbs:
      - update
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      hostNetwork: true
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                # 指定部署到三台 master 上
                values:
                - tf-k8s-m1
                - tf-k8s-m2
                - tf-k8s-m3
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/name
                    operator: In
                    values: 
                    - ingress-nginx
              topologyKey: "kubernetes.io/hostname"
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: registry.cn-hangzhou.aliyuncs.com/google_containers/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=/nginx-configuration
            - --tcp-services-configmap=/tcp-services
            - --udp-services-configmap=/udp-services
            # - --publish-service=/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 1
          resources:
            limits:
              cpu: 1
              memory: 1024Mi
            requests:
              cpu: 0.25
              memory: 512Mi
```
部署 nginx ingress，执行命令 kubectl apply -f nginx-ingress.yaml

### 2.9 部署kubernetes-dashboard
#### 2.9.1 Dashboard 配置
新建部署 dashboard 的资源配置文件：kubernetes-dashboard.yaml，内容如下：

```
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create 'kubernetes-dashboard-key-holder' secret.
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create"]
  # Allow Dashboard to create 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs"]
  verbs: ["get", "update", "delete"]
  # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kubernetes-dashboard-settings"]
  verbs: ["get", "update"]
  # Allow Dashboard to get metrics from heapster.
- apiGroups: [""]
  resources: ["services"]
  resourceNames: ["heapster"]
  verbs: ["proxy"]
- apiGroups: [""]
  resources: ["services/proxy"]
  resourceNames: ["heapster", "http:heapster:", "https:heapster:"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard-minimal
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
---
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
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
        # 使用阿里云的镜像
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.0
        ports:
        - containerPort: 8443
          protocol: TCP
        args:
          - --auto-generate-certificates
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443
          initialDelaySeconds: 30
          timeoutSeconds: 30
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
---
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
---
# 配置 ingress 配置，待会部署完 ingress 之后，就可以通过以下配置的域名访问
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kube-system
  annotations:
    # 如果通过 HTTP 访问，跳转到 HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    # 指定转发协议为 HTTPS，因为 ingress 默认转发协议是 HTTP，而 kubernetes-dashboard 默认是 HTTPS
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  # 指定使用的 secret (刚刚创建的 secret)
  tls:
   - secretName: secret-ca-tf-k8s-xiangwushuo-com
  rules:
  # 指定访问 dashboard 的域名
  - host: dashboard.tf-k8s.xiangwushuo.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```
执行部署 kubernetes-dashboard，命令 kubectl apply -f kubernetes-dashboard.yaml.

在本地笔记本电脑上访问dashboard的时候，需要将dashboard.tf-k8s.xiangwushuo.com域名解析到三台master的IP（配置代理），简单地，可以直接在本地/etc/hosts添加

```
## 172.66.23.13 为tf-k8s-m1的外网IP
172.66.23.13 dashboard.tf-k8s.xiangwushuo.com
```
从浏览器访问: http://dashboard.tf-k8s.xiangwushuo.com
![](/images/dashboard-login.png)


#### 2.9.2 HTTPS 访问 Dashboard
由于通过 HTTP 访问 dashboard 会无法登录进去 dashboard 的问题，所以这里我们将 dashboard 的服务配置成 HTTPS 进行访问。
总共三步:
签证书（或者使用权威的证书机构颁发的证书）

```
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout ./tf-k8s.xiangwushuo.com.key -out ./tf-k8s.xiangwushuo.com.crt -subj "/CN=*.xiangwushuo.com"
```

创建 k8s Secret 资源

```
kubectl -n kube-system create secret tls secret-ca-tf-k8s-xiangwushuo-com --key ./tf-k8s.xiangwushuo.com.key --cert tf-k8s.xiangwushuo.com.crt 
```

配置 dashboard 的 ingress 为 HTTPS 访问服务,修改 kubernetes-dashboard.yaml，将其中的 Ingress 配置改为支持 HTTPS，具体配置如下：

```
...省略...

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kube-system
  annotations:
    # 如果通过 HTTP 访问，跳转到 HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    # 指定转发协议为 HTTPS，因为 ingress 默认转发协议是 HTTP，而 kubernetes-dashboard 默认是 HTTPS
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  # 指定使用的 secret (刚刚创建的 secret)
  tls:
   - secretName: secret-ca-k8s-hiko-im
  rules:
  # 指定访问 dashboard 的域名
  - host: dashboard.k8s.hiko.im
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443

```
使用 kubectl apply -f kubernetes-dashboard.yaml 让配置生效。

#### 2.9.3 .3 登录 Dashboard
登录 dashboard 需要做几个事情（不用担心，一个脚本搞定）:

新建 sa 的账号（也叫 serviceaccount）
集群角色绑定（将第 1 步新建的账号，绑定到 cluster-admin 这个角色上）
查看 Token 以及 Token 中的 secrect （secrect 中的 token 字段就是来登录的）
执行以下脚本，获得登录的 Token:

```

## 创建脚本：create.dashboard.token.sh

#!/bin/sh

kubectl create sa dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep dashboard-admin | awk '{print $1}')
DASHBOARD_LOGIN_TOKEN=$(kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}')
echo ${DASHBOARD_LOGIN_TOKEN}
```

复制 Token 去登录就行，Token 样例：

```

eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tNWtnZHoiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYWQxNDAyMjQtMDYxNC0xMWU5LTkxMDgtNTI1NDAwODQ4MWQ1Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.ry4xYI6TFF6J8xXsilu0qhuBeRjSNqVPq3OUzl62Ad3e2wM-biC5pPlKNmJLfJzurxnQrqp59VjmVeTA8BZiF7S6hqlrk8XE9_LFlItUvq3rp5wFuhJuVol8Yoi4UJFzUYQF6baH0O3R10aK33g2WmWLIg79OFAkeMMHrLthbL2pc_p_kG13_qDXlEuVgnIAFsKzxnrCCUfZ2GwGsHEFEqTGBCb0u6x3AZqfQgbN3DALkjjNTyTLP5Ok-LJ3Ug8SZZQBksvTeXCGXZDfk2LDDIvp_DyM7nTL3CTT5cQ3g4aBTFAae47NAkQkmjZg0mxvJH0xVnxrvXLND8FLLkzMxg
```

## 3. 参考文献
[1. kubeadm 1.13 安装高可用 kubernetes v1.13.1 集群](https://www.ctolib.com/HikoQiu-kubeadm-install-k8s.html)
[2. 如何在CentOS 7上修改主机名](https://www.jianshu.com/p/39d7000dfa47)
[3. Linux之ssh免密登录](https://blog.csdn.net/mmd0308/article/details/73825953)
[4. sudo与visudo的超细用法说明](http://blog.51cto.com/chenfage/1830424)
[5. kubeadm HA master(v1.13.0)离线包 + 自动化脚本 + 常用插件 For Centos/Fedora](https://www.kubernetes.org.cn/4948.html)
[6. github.coreos.flannel](https://github.com/coreos/flannel)
[7. Kubernetes Handbook——Kubernetes中文指南/云原生应用架构实践手册](https://jimmysong.io/kubernetes-handbook/)

