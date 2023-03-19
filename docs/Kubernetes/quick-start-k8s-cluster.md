# 在VMWare中部署你的K8S集群

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

- `Docker版本: Docker version 18.06.1-ce, build e68fc7a`
- `K8S版本: 1.18.0`
- `三个节点: master、node1、node2（固定IP）`
- `容器运行时: 仍然使用Docker而非Containerd`
- `Pod网络: 用Calico替换Flannel实现 Pod 互通，支持更大规模的集群`
- `集群构建工具: Kubeadm`

大概的架构就是类似下面找的图这种:

 ![kubernete](https://blog-1252881505.cos.ap-beijing.myqcloud.com/k8s/single-master.jpg) 

关于网络配置

- 整体机器采用NAT地址转换
- 各台虚拟机采用固定IP地址
- 虚拟机VMWare统一网关地址：192.168.148.2

## 学习目标

1. 在所有节点上安装Docker和kubeadm
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

```
关闭防火墙：
$ systemctl stop firewalld
$ systemctl disable firewalld

关闭selinux：
$ sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
$ setenforce 0  # 临时

关闭swap：
$ swapoff -a  # 临时
$ vim /etc/fstab  # 永久

设置主机名：
$ hostnamectl set-hostname <hostname>

在master添加hosts：
$ cat >> /etc/hosts << EOF
192.168.31.61 master
192.168.31.62 node1
192.168.31.63 node2
EOF

将桥接的IPv4流量传递到iptables的链：
$ cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
$ sysctl --system  # 生效

时间同步：
$ yum install ntpdate -y
$ ntpdate time.windows.com
```

## 所有节点安装Docker/kubeadm/kubelet

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

### 安装Docker

```
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
$ yum -y install docker-ce-18.06.1.ce-3.el7
$ systemctl enable docker && systemctl start docker
$ docker --version
Docker version 18.06.1-ce, build e68fc7a
```

```
# cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

### 添加阿里云YUM软件源

```
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：

```
$ yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
$ systemctl enable kubelet
```

## 部署Kubernetes Master

在192.168.148.61（Master）执行。

```
$ kubeadm init \
  --apiserver-advertise-address=192.168.148.61 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.18.0 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16
```

由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。

使用kubectl工具：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl get nodes
```

## 加入Kubernetes Node

在192.168.148.62/63（Node）执行。

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```
$ kubeadm join 192.168.148.61:6443 --token 681ntf.gfq6b9o3zdrulfwu \
    --discovery-token-ca-cert-hash sha256:9b2c0f1f39484dd99fd67e5e2614c14ae6a59d44eb1e8dff81e3de3788afd8ff
```

默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：

```
# kubeadm token create
# kubeadm token list
# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
63bca849e0e01691ae14eab449570284f0c3ddeea590f8da988c07fe2729e924

# kubeadm join 192.168.148.61:6443 --token nuja6n.o3jrhsffiqs9swnu --discovery-token-ca-cert-hash sha256:63bca849e0e01691ae14eab449570284f0c3ddeea590f8da988c07fe2729e924
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

 <https://docs.projectcalico.org/getting-started/kubernetes/quickstart> 

```bash
kubectl apply -f https://docs.projectcalico.org/archive/v3.13/manifests/calico.yaml
```

下载完后还需要修改里面配置项：

- 根据实际网络规划修改Pod CIDR（CALICO_IPV4POOL_CIDR）
- 选择工作模式（CALICO_IPV4POOL_IPIP），支持**BGP（Never）**、**IPIP（Always）**、**CrossSubnet**（开启BGP并支持跨子网）

修改完后应用清单：

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