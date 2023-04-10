# 在VMWare中部署你的K8S(1.18.0)集群

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

这个工具能通过两条指令完成一个kubernetes集群的部署：

```
# 创建一个 Master 节点
$ kubeadm init

# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口 >
```

## 安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- `一台或多台机器，操作系统 CentOS7.x-86_x64`
- `硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多`
- `集群中所有机器之间网络互通`
- `可以访问外网，需要拉取镜像`
- `禁止swap分区`

我个人的机器配置:

- `CPU: AMD Ryzen 9 5900X`
- `内存: 32G`
- `硬盘: KIOXIA-EXCERIA PLUS G2 SSD`
- `虚拟化: VMWare Pro 16.0.0 build-16894299`
- `操作系统镜像: CentOS-7-x86_64-Minimal-1810.iso`

搭建的K8S集群信息

- `Docker版本(制作镜像): Docker version 23.0.2, build 569dd73`
- `K8S版本: v1.25.0`
- `三个节点: master、node1、node2`
- `容器运行时: Containerd`
- `Pod网络: 用Calico替换Flannel实现 Pod 互通，支持更大规模的集群`
- `集群构建工具: Kubeadm`

大概的架构就是类似下面找的图这种:

 ![kubernete](https://blog-1252881505.cos.ap-beijing.myqcloud.com/k8s/single-master.jpg) 

关于网络配置

- 整体机器采用NAT地址转换
- 各台虚拟机采用固定IP地址
- 虚拟机VMWare统一网关地址：192.168.148.2

## 学习目标

1. 在所有节点上安装Docker和kubeadm, Containerd
2. 部署Kubernetes Master
3. 部署容器网络插件
4. 部署 Kubernetes Node，将节点加入Kubernetes集群中
5. 部署Dashboard Web页面，可视化查看Kubernetes资源

## 虚拟环境搭建

[centos7镜像地址](http://ftp.iij.ad.jp/pub/linux/centos-vault/7.6.1810/isos/x86_64/)

虚拟机配置信息

- 1个处理器4核
- 4G内存
- 40G硬盘SCSI
- 网络：NAT

## 环境配置

修改配置`vi /etc/sysconfig/network`

```bash
vi /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=master
```

修改配置`vi /etc/sysconfig/network-scripts/ifcfg-ens33`

```bash
vi /etc/sysconfig/network-scripts/ifcfg-ens33
# 配置如下

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
ONBOOT=yes
IPADDR=192.168.148.61
NETMASK=255.255.255.0
GATEWAY=192.168.148.2
NAME=ens33
DEVICE=ens33
DNS1=192.168.148.2
DNS2=8.8.8.8
```

上面的配置主要是为了将网络修改为静态IP,配置重点:

- `BOOTPROTO：修改为static，即静态IP`
- `IPADDR：静态的IP地址`
- `NETMASK：子网掩码，选择255.255.255.0即可`
- `GATEWAY：网关，结尾为.2，NAT分配的子网地址可在编辑→虚拟网络编辑器中查看`
- `DNS1、DNS2：DNS1可选择本机网关地址、DNS2可配置114.114.114.114等公共DNS`
- `ONBOOT：yes，开机启用网络（开启才能开机就联网）`

| 角色       | IP            |
| ---------- | ------------- |
| master | 192.168.148.61 |
| node1  | 192.168.148.62 |
| node2  | 192.168.148.63 |
| gateway  | 192.168.148.2 |

`关闭防火墙：`

```bash
systemctl stop firewalld  && systemctl disable firewalld && systemctl status firewalld
```

`关闭selinux：`

```bash
#修改 selinux 配置文件之后，重启机器，selinux 配置才能永久生效重启之后登录机器验证是否修改成功：
#显示 Disabled 说明 selinux 已经关闭
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config && getenforce
```

`关闭swap：`

```bash
#临时关闭 
swapoff -a
#永久关闭：注释 swap 挂载，给 swap 这行开头加一下注释
vim /etc/fstab
#dev/mapper/centos-swap swap	swap	defaults	0 0
```

`转发 IPv4 并让 iptables 看到桥接流量：`

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

# 通过运行以下指令确认 br_netfilter 和 overlay 模块被加载
lsmod | grep br_netfilter
lsmod | grep overlay

# 通过运行以下指令确认 net.bridge.bridge-nf-call-iptables、net.bridge.bridge-nf-call-ip6tables 和 net.ipv4.ip_forward 系统变量在你的 sysctl 配置中被设置为 1：
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

`配置阿里云 repo 源：`

```bash
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

`时间同步：`

```bash
#安装 ntpdate 或chrony服务都可以
yum install ntpdate -y
#跟网络源做同步
ntpdate cn.pool.ntp.org
#把时间同步做成计划任务 
crontab -e
* */1 * * * /usr/sbin/ntpdate	cn.pool.ntp.org
#重启 crond 服务
service crond restart
```

`安装环境基础软件包：`

```bash
yum update 
yum install -y device-mapper-persistent-data lvm2 wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim ncurses-devel autoconf automake zlib-devel python-devel epel-release openssh-server socat  ipvsadm conntrack ntpdate telnet rsync
```

`设置主机名：`

```bash
hostnamectl set-hostname <hostname>
```

`添加hosts：`

```bash
cat >> /etc/hosts << EOF
192.168.148.61 master
192.168.148.62 node1
192.168.148.63 node2
EOF
```

## 所有节点安装Docker/kubeadm/kubelet/containerd

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

### 安装 containerd 及 docker 服务

`安装containerd`

```bash
yum install containerd.io-1.6.6 -y
```

`配置containerd配置`

```bash
mkdir -p /etc/containerd
# 生成containerd配置文件
containerd config default > /etc/containerd/config.toml
# 修改配置文件
# 把SystemdCgroup = false修改成SystemdCgroup = true
# 把sandbox_image = "k8s.gcr.io/pause:3.6"修改成sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.7"
vim /etc/containerd/config.toml
```

`配置 containerd 开机启动，并启动 containerd`

```bash
systemctl enable containerd  --now
```

`修改/etc/crictl.yaml文件并重启`

```bash
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

systemctl restart containerd
```

`配置containerd镜像加速器`

```bash
# 将config_path = ""修改成如下目录：config_path = "/etc/containerd/certs.d"
vim /etc/containerd/config.toml文件
mkdir /etc/containerd/certs.d/docker.io/ -p
# 添加配置
# [host."https://vh3bm52y.mirror.aliyuncs.com",host."https://registry.docker-cn.com"]
#   capabilities = ["pull"] 
vim /etc/containerd/certs.d/docker.io/hosts.toml
systemctl restart containerd
```

`安装 docker 环境`

PS: docker也要安装，docker跟containerd不冲突，安装docker是为了能基于dockerfile构建镜像

```bash
yum install docker-ce -y
systemctl enable docker --now
```

`配置 docker 镜像加速器`

```bash
tee /etc/docker/daemon.json << 'EOF'
{
"registry-mirrors":["https://vgsnzrqa.mirror.aliyuncs.com","https://registry.docker-cn.com","https://docker.mirrors.ustc.edu.cn","https://dockerhub.azk8s.cn","http://hub-mirror.c.163.com","http://qtid6917.mirror.aliyuncs.com", "https://rncxm540.mirror.aliyuncs.com"],
"exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
systemctl daemon-reload && systemctl restart docker && systemctl status docker
```

### 安装kubeadm，kubelet和kubectl

`配置安装k8s组件需要的阿里云的repo源`

```bash
# 修改下面内容
# [kubernetes]
# name=Kubernetes
# baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
# enabled=1
# gpgcheck=0
vim /etc/yum.repos.d/kubernetes.repo
```

由于版本更新频繁，这里指定版本号部署：

```
$ yum install -y kubelet-1.25.0 kubeadm-1.25.0 kubectl-1.25.0
$ systemctl enable kubelet
```

`设置容器运行时的endpoing`

```bash
crictl config runtime-endpoint /run/containerd/containerd.sock
```

## 部署Kubernetes Master

在192.168.148.61（Master）执行。

`生成集群 kubeadm 配置文件`

```bash
kubeadm config print init-defaults > kubeadm.yaml
# 根据我们自己的需求修改配置，比如修改 imageRepository 的值，kube-proxy 的模式为 ipvs，需要注意的是由于我们使用的containerd作为运行时，所以在初始化节点的时候需要指定cgroupDriver为systemd
# 由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。
cat kubeadm.yaml
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
  advertiseAddress: 192.168.148.61 #控制节点的ip
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock #指定containerd容器运行时的endpoint
  imagePullPolicy: IfNotPresent
  name: node #控制节点主机名
  taints: null
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
imageRepository: registry.aliyuncs.com/google_containers #指定从阿里云仓库拉取镜像
kind: ClusterConfiguration
kubernetesVersion: 1.25.0 #k8s版本
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16 #指定pod网段
  serviceSubnet: 10.96.0.0/12 #指定Service网段
scheduler: {}
# 下面的属于自己添加的部分
# kube-proxy 的模式为 ipvs
# 指定cgroupDriver为systemd
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

`基于kubeadm.yaml文件初始化k8s`

```bash
sudo kubeadm init --config=kubeadm.yaml --ignore-preflight-errors=SystemVerification
```

`配置kubectl的配置文件`

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl get nodes
```

## 加入Kubernetes Node

`在 master1 上查看加入节点的命令`

```bash
kubeadm token create --print-join-command
kubeadm join 192.168.148.61:6443 --token 1h5lo4.sowycilwve5u7k7b --discovery-token-ca-cert-hash sha256:c3218416a8097dd0ddc2ffba794fd7bbb8c8969f9845aa5ae7e042b9022d844f
```

`把 node1 和 node2 加入k8s集群`
在 node1 和 node2 执行

```bash
kubeadm join 192.168.148.61:6443 --token 1h5lo4.sowycilwve5u7k7b --discovery-token-ca-cert-hash sha256:c3218416a8097dd0ddc2ffba794fd7bbb8c8969f9845aa5ae7e042b9022d844f
```

`默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：`

```bash
kubeadm token create
kubeadm token list
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
63bca849e0e01691ae14eab449570284f0c3ddeea590f8da988c07fe2729e924

kubeadm join 192.168.148.61:6443 --token nuja6n.o3jrhsffiqs9swnu --discovery-token-ca-cert-hash sha256:63bca849e0e01691ae14eab449570284f0c3ddeea590f8da988c07fe2729e924
```

kubeadm token create --print-join-command

<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/>

## 网络方案（CNI）

 <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network> 

### Flannel

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
sed -i -r "s#quay.io/coreos/flannel:.*-amd64#lizhenliang/flannel:v0.11.0-amd64#g" kube-flannel.yml
kubectl apply -f ./kube-flannel.yml
```

修改国内镜像仓库。

删除 Flannel 网络插件和对用数据

```bash
kubectl delete -f ./kube-flannel.yml
ip link del cni0
ip link del flannel.1
```

### Calico

`官网地址安装教程地址`

 <https://docs.projectcalico.org/getting-started/kubernetes/quickstart> 

 `下載配置文件`

```bash
wget https://projectcalico.docs.tigera.io/archive/v3.25/manifests/calico.yaml
```

`配置网卡名`

```bash
# calico 默认找 eth0 网卡,如果当前机器不是这个名字,建议手动配置,流程如下:
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:48:7b:d7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.148.61/24 brd 192.168.148.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::56fc:402d:4f4:3717/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:c8:9e:eb:49 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 26:d6:43:e7:d0:84 brd ff:ff:ff:ff:ff:ff

# 这里我的网卡名是 ens33
vi calico.yaml
# 搜索 CLUSTER_TYPE
# - name: CLUSTER_TYPE
#   value: "k8s,bgp"
# 添加同层级字段 IP_AUTODETECTION_METHOD
# value 就是指定你的网卡名字，我这里网卡是 ens33，然后直接配置的通配符 ens.*
# - name: IP_AUTODETECTION_METHOD
#   value: "interface=ens.*"
```

下载完后还需要修改里面配置项：

- 根据实际网络规划修改Pod CIDR（CALICO_IPV4POOL_CIDR）
- 选择工作模式（CALICO_IPV4POOL_IPIP），支持 **BGP（Never）**、**IPIP（Always）**、**CrossSubnet**（开启BGP并支持跨子网）

`执行部署：`

```bash
kubectl apply -f calico.yaml
kubectl get pods -n kube-system
```

## 测试kubernetes集群

- 能不能正常部署应用: 验证Pod工作
- 集群网络是否正常: 验证Pod网络通信
- 集群内部dns解析是否正常: 验证DNS解析

在Kubernetes集群中创建一个pod，验证是否正常运行：

```
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --type=NodePort
$ kubectl get pod,svc
```

访问地址：<http://NodeIP:Port>  

## 部署 Dashboard

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

默认Dashboard只能集群内部访问，修改Service为NodePort类型，暴露到外部：

```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
```

访问地址：<https://NodeIP:30001>

创建service account并绑定默认cluster-admin管理员集群角色：

```bash
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```

使用输出的token登录Dashboard。

### 解决Dashboard其他浏览器不能访问

#### 二进制 部署

`注意你部署Dashboard的命名空间（之前部署默认是kube-system，新版是kubernetes-dashboard）`

1. `删除默认的secret，用自签证书创建新的secret`

```bash
kubectl delete secret kubernetes-dashboard-certs -n kubernetes-dashboard
kubectl create secret generic kubernetes-dashboard-certs \
--from-file=/opt/kubernetes/ssl/server-key.pem --from-file=/opt/kubernetes/ssl/server.pem -n kubernetes-dashboard
```

2. 修改 dashboard.yaml 文件，在args下面增加证书两行

```yml
  args:
    # PLATFORM-SPECIFIC ARGS HERE
    - --auto-generate-certificates
    - --tls-key-file=server-key.pem
    - --tls-cert-file=server.pem
```

```bash
kubectl apply -f dashboard.yaml.yaml
```

#### kubeadm 部署

`注意你部署Dashboard的命名空间（之前部署默认是kube-system，新版是kubernetes-dashboard）`

1. 删除默认的secret，用自签证书创建新的secret

```bash
kubectl delete secret kubernetes-dashboard-certs -n kubernetes-dashboard
kubectl create secret generic kubernetes-dashboard-certs \
--from-file=/etc/kubernetes/pki/apiserver.key --from-file=/etc/kubernetes/pki/apiserver.crt -n kubernetes-dashboard
```

2. 修改 dashboard.yaml 文件，在args下面增加证书两行

```yml
  args:
    # PLATFORM-SPECIFIC ARGS HERE
    - --auto-generate-certificates
    - --tls-key-file=apiserver.key
    - --tls-cert-file=apiserver.crt
```

```bash
kubectl apply -f dashboard.yaml.yaml
```