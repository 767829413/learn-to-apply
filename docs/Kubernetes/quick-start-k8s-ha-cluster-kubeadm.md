# 在VMWare中部署你的K8S(1.25.0)集群

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

这个工具能通过两条指令完成一个kubernetes集群的部署：

```
# 创建一个 Master 节点
$ kubeadm init

# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口 >
```

## 安装要求 && 环境配置 && 虚拟环境搭建

可以借鉴:

[在VMWare中部署你的K8S(1.25.0)集群-kubeadm](../../docs/Kubernetes/quick-start-k8s-cluster-kubeadm.md)

## 整体规划

`架构图`

![架构图](https://pic.imgdb.cn/item/643519f00d2dde57776e975f.png)

`软件环境`

| name | version |
| --- | --- |
| 操作系统 | CentOS7.9_x64 （mini） |
| Containerd | 1.6.6 |
| Docker | 23.0.3-ce |
| Kubernetes | 1.25.0 |

`服务器角色`

| 角色 | IP | 其他单装组件 |
| --- | --- | --- |
| master-1 | 192.168.148.70 | docker/kubeadm/kubelet/containerd/nginx/keepalived/etcd |
| master-2 | 192.168.148.71 | kubeadm/kubelet/containerd/nginx/keepalived/etcd |
| node-1 | 192.168.148.72 | kubeadm/kubelet/containerd/etcd |
| 负载均衡器对外IP | 192.168.148.73 |  |

## 部署Nginx+Keepalived高可用负载均衡器

`介绍`

```text
Kubernetes作为容器集群系统，通过健康检查+重启策略实现了Pod故障自我修复能力，通过调度算法实现将Pod分布式部署，并保持预期副本数，根据Node失效状态自动在其他Node拉起Pod，实现了应用层的高可用性。 
针对Kubernetes集群，高可用性还应包含以下两个层面的考虑：Etcd数据库的高可用性和Kubernetes Master组件的高可用性。 而kubeadm搭建的K8s集群，Etcd只起了一个，存在单点，所以我们这里会独立搭建一个Etcd集群。
Master节点扮演着总控中心的角色，通过不断与工作节点上的Kubelet和kube-proxy进行通信来维护整个集群的健康工作状态。如果Master节点故障，将无法使用kubectl工具或者API做任何集群管理。
Master节点主要有三个服务kube-apiserver、kube-controller-manager和kube-scheduler，其中kube-controller-manager和kube-scheduler组件自身通过选举机制已经实现了高可用，所以Master高可用主要针对kube-apiserver组件，而该组件是以HTTP API提供服务，因此对他高可用与Web服务器类似，增加负载均衡器对其负载均衡即可，并且可水平扩容
```

`kube-apiserver高可用架构图`

![架构图](https://pic.imgdb.cn/item/643521830d2dde57777bebd2.png)

- Nginx是一个主流Web服务和反向代理服务器，这里用四层实现对apiserver实现负载均衡
- Keepalived是一个主流高可用软件，基于VIP绑定实现服务器双机热备，在上述拓扑中，Keepalived主要根据Nginx运行状态判断是否需要故障转移（偏移VIP），例如当Nginx主节点挂掉，VIP会自动绑定在Nginx备节点，从而保证VIP一直可用，实现Nginx高可用

`安装软件包（主/备）`

```bash
yum install epel-release -y
yum install nginx keepalived -y
yum install nginx-mod-stream -y
```

`Nginx配置文件（主/备一致）`

```bash
cat > /etc/nginx/nginx.conf << "EOF"
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

# 四层负载均衡，为两台Master apiserver组件提供负载均衡
stream {

    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';

    access_log  /var/log/nginx/k8s-access.log  main;

    # 填写自己的ip地址
    upstream k8s-apiserver {
       server 192.168.148.70:6443;   # Master1 APISERVER IP:PORT
       server 192.168.148.71:6443;   # Master2 APISERVER IP:PORT
    }

    server {
       listen 16443;  # 由于nginx与master节点复用，这个监听端口不能是6443，否则会冲突
       proxy_pass k8s-apiserver;
    }
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;
}
EOF
```

`keepalived配置文件（Nginx Master-1）`

- vrrp_script：指定检查nginx工作状态脚本（根据nginx状态判断是否故障转移）
- virtual_ipaddress：虚拟IP（VIP）

```bash
cat > /etc/keepalived/keepalived.conf << EOF
global_defs { 
   notification_email { 
     acassen@firewall.loc 
     failover@firewall.loc 
     sysadmin@firewall.loc 
   } 
   notification_email_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1 
   smtp_connect_timeout 30 
   router_id NGINX_MASTER
} 

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"# 判断返回状态码
}

vrrp_instance VI_1 { 
    state MASTER 
    interface ens33  # 修改为实际网卡名 ip a 查看
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
    priority 100    # 优先级，备服务器设置 90 
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒 
    authentication { 
        auth_type PASS      
        auth_pass 1111 
    }  
    # 虚拟IP
    virtual_ipaddress { 
        192.168.148.73/24
    } 
    track_script {
        check_nginx
    } 
}
EOF
```

`准备上述配置文件中检查nginx运行状态的脚本`

```bash
cat > /etc/keepalived/check_nginx.sh  << "EOF"
#!/bin/bash
count=$(ss -antp | grep 16443 | egrep -cv "grep|$$")

if [ "$count" -eq 0 ];then
    exit 1
else
    exit 0
fi
EOF
chmod +x /etc/keepalived/check_nginx.sh
```

`keepalived配置文件（Nginx master-2）`

```bash
cat > /etc/keepalived/keepalived.conf << EOF
global_defs { 
   notification_email { 
     acassen@firewall.loc 
     failover@firewall.loc 
     sysadmin@firewall.loc 
   } 
   notification_email_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1 
   smtp_connect_timeout 30 
   router_id NGINX_BACKUP
} 

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}

vrrp_instance VI_1 { 
    state BACKUP 
    interface ens33
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的 
    priority 90
    advert_int 1
    authentication { 
        auth_type PASS      
        auth_pass 1111 
    }  
    virtual_ipaddress { 
        192.168.148.73/24
    } 
    track_script {
        check_nginx
    } 
}
EOF
```

`准备上述配置文件中检查nginx运行状态的脚本`

```bash
cat > /etc/keepalived/check_nginx.sh  << "EOF"
#!/bin/bash
count=$(ss -antp |grep 16443 |egrep -cv "grep|$$")

if [ "$count" -eq 0 ];then
    exit 1
else
    exit 0
fi
EOF
chmod +x /etc/keepalived/check_nginx.sh
```

**PS: keepalived根据脚本返回状态码（0为工作正常，非0不正常）判断是否故障转移**

`启动并设置开机启动(master-1 master-2)`

```bash
systemctl daemon-reload
systemctl start nginx
systemctl start keepalived
systemctl enable nginx
systemctl enable keepalived
```

`查看keepalived工作状态`

```bash
# 可以看到，在ens33网卡绑定了 192.168.148.73 虚拟IP，说明工作正常
ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:09:04:81 brd ff:ff:ff:ff:ff:ff
    inet 192.168.148.70/24 brd 192.168.148.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.148.73/24 scope global secondary ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::d9d2:39d6:66e3:b9fc/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
    inet6 fe80::a3ff:e4d5:8f6e:4a4c/64 scope link tentative noprefixroute dadfailed 
       valid_lft forever preferred_lft forever
    inet6 fe80::3183:f57c:c91:aa21/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

`Nginx+Keepalived高可用测试`

```bash
# 关闭主节点Nginx，测试VIP是否漂移到备节点服务器
# 在Nginx master-1 执行
pkill nginx
# 在Nginx master-2，ip addr命令查看已成功绑定VIP
ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:91:0b:67 brd ff:ff:ff:ff:ff:ff
    inet 192.168.148.71/24 brd 192.168.148.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.148.73/24 scope global secondary ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::d9d2:39d6:66e3:b9fc/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

## 部署Etcd集群

`基本介绍`

```text
Etcd 是一个分布式键值存储系统，Kubernetes使用Etcd进行数据存储，kubeadm搭建默认情况下只启动一个Etcd Pod，存在单点故障，生产环境强烈不建议，所以我们这里使用3台服务器组建集群，可容忍1台机器故障，当然，你也可以使用5台组建集群，可容忍2台机器故障。
```

`节点信息`

| 节点名称 | IP |
| --- | --- |
| etcd-1 | 192.168.148.70 |
| etcd-2 | 192.168.148.71 |
| etcd-3 | 192.168.148.72 |

**PS: 为了节省机器，这里与 K8S 节点机器复用。也可以独立于 K8S 集群之外部署，只要 apiserver 能连接到就行**

### 准备cfssl证书生成工具

**cfssl是一个开源的证书管理工具，使用json文件生成证书，相比openssl更方便使用,找任意一台服务器操作，这里用 Master-1 节点**

```bash
mkdir ~/cfssl && cd ~/cfssl
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl-certinfo_1.6.4_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64
chmod +x ./*
mv cfssl_1.6.4_linux_amd64 /usr/local/bin/cfssl
mv cfssl-certinfo_1.6.4_linux_amd64 /usr/local/bin/cfssl-certinfo
mv cfssljson_1.6.4_linux_amd64 /usr/bin/cfssljson
cd ../ && rm -fr ~/cfssl
```

### 生成Etcd证书

`自签证书颁发机构（CA）`

```bash
mkdir -p ~/etcd_tls && cd ~/etcd_tls
# 自签CA
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
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

cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

# 生成证书
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```

`使用自签CA签发Etcd HTTPS证书`

```bash
# 创建证书申请文件
# 文件 hosts 字段中 IP 为所有 etcd 节点的集群内部通信 IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP
cat > server-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "192.168.148.70",
    "192.168.148.71",
    "192.168.148.72",
    "192.168.148.75",
    "192.168.148.76",
    "192.168.148.77",
    "192.168.148.78",
    "192.168.148.79"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem  server.csr  server-csr.json  server-key.pem  server.pem
```

### ETCD部署操作

`从Github下载二进制文件`

```bash
cd ~
wget https://github.com/etcd-io/etcd/releases/download/v3.5.7/etcd-v3.5.7-linux-amd64.tar.gz
```

`预创建Etcd相关文件夹`

```bash
mkdir /opt/etcd/{bin,cfg,ssl} -p
tar zxvf etcd-v3.5.7-linux-amd64.tar.gz
ls
etcd_tls  etcd-v3.5.7-linux-amd64  etcd-v3.5.7-linux-amd64.tar.gz
mv etcd-v3.5.7-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```

`创建etcd配置文件`

- ETCD_NAME：节点名称，集群中唯一
- ETCD_DATA_DIR：数据目录
- ETCD_LISTEN_PEER_URLS：集群通信监听地址
- ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
- ETCD_INITIAL_ADVERTISEPEERURLS：集群通告地址
- ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
- ETCD_INITIAL_CLUSTER：集群节点地址
- ETCD_INITIAL_CLUSTER_TOKEN：集群Token
- ETCD_INITIAL_CLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群

```bash
cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.148.70:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.148.70:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.148.70:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.148.70:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.148.70:2380,etcd-2=https://192.168.148.71:2380,etcd-3=https://192.168.148.72:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

`systemd管理etcd`

```bash
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

`证书位置对应`

```bash
cp ~/etcd_tls/ca*pem ~/etcd_tls/server*pem /opt/etcd/ssl/
```

`将上面 master-1 所有生成的文件拷贝到 master-2 和 node-1`

```bash
scp -r /opt/etcd/ root@192.168.148.71:/opt/
scp /usr/lib/systemd/system/etcd.service root@192.168.148.71:/usr/lib/systemd/system/
scp -r /opt/etcd/ root@192.168.148.72:/opt/
scp /usr/lib/systemd/system/etcd.service root@192.168.148.72:/usr/lib/systemd/system/
```

`master-2 和 node-1 分别修改 etcd.conf 配置文件中的节点名称和当前服务器 IP`

```bash
# 这里以 master-2 举例
# node-1 修改类似
vi /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-1"   # 修改此处，master-2 改为 etcd-2，node-1 改为 etcd-3
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.148.71:2380"   # 修改此处为当前服务器IP
ETCD_LISTEN_CLIENT_URLS="https://192.168.148.71:2379" # 修改此处为当前服务器IP

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.148.71:2380" # 修改此处为当前服务器IP
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.148.71:2379" # 修改此处为当前服务器IP
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.148.70:2380,etcd-2=https://192.168.148.71:2380,etcd-3=https://192.168.148.72:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

`master-1 和 master-2, node-1 启动 Etcd`

**PS: 可以先启动 master-2, node-1 再启动 master-1**

```bash
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd
```

`查看集群状态`

```bash
ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.148.70:2379,https://192.168.148.71:2379,https://192.168.148.72:2379" endpoint health --write-out=table
+-----------------------------+--------+-------------+-------+
|          ENDPOINT           | HEALTH |    TOOK     | ERROR |
+-----------------------------+--------+-------------+-------+
| https://192.168.148.70:2379 |   true |  7.460363ms |       |
| https://192.168.148.71:2379 |   true |  8.864573ms |       |
| https://192.168.148.72:2379 |   true | 11.000779ms |       |
+-----------------------------+--------+-------------+-------+
```

## 所有节点安装 kubeadm/kubelet/containerd

**PS:master-1 节点额外安装一个 Docker 用来构建镜像**

### 安装 containerd

`安装containerd`

```bash
yum install containerd.io-1.6.6 -y
```

`containerd配置`

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
vim /etc/containerd/config.toml
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

在192.168.148.70（Master-1）执行。

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
  advertiseAddress: 192.168.148.70 #控制节点的ip
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock #指定containerd容器运行时的endpoint
  imagePullPolicy: IfNotPresent
  name: master-1 #控制节点主机名
  taints: 
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs: #包含所有Master/LB/VIP IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP
  - master-1
  - master-2
  - 192.168.148.70
  - 192.168.148.71
  - 192.168.148.72
  - 127.0.0.1
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 192.168.148.73:16443 #负载均衡虚拟IP（VIP）和端口
controllerManager: {}
dns: {}
etcd:
  external: #使用外部etcd
    endpoints:
    - https://192.168.148.70:2379 #etcd集群3个节点
    - https://192.168.148.71:2379
    - https://192.168.148.72:2379
    caFile: /opt/etcd/ssl/ca.pem #连接etcd所需证书
    certFile: /opt/etcd/ssl/server.pem
    keyFile: /opt/etcd/ssl/server-key.pem
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
sudo kubeadm init --config=kubeadm.yaml

...
You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join 192.168.148.73:16443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:78c0233ca7d519a9faab74bf55ce194eb9aba2dcbcb578d29614b979421e7424 \
	--control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.148.73:16443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:78c0233ca7d519a9faab74bf55ce194eb9aba2dcbcb578d29614b979421e7424
```

**PS:初始化完成后，会有两个join的命令，带有 --control-plane 是用于加入组建多master集群的，不带的是加入节点的**

`配置kubectl的配置文件`

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
NAME       STATUS     ROLES           AGE    VERSION
master-1   NotReady   control-plane   103s   v1.25.0
```

## 初始化 master-2

`将 master-1 节点生成的证书拷贝到 master-2`

```bash
scp -r /etc/kubernetes/pki/ 192.168.148.71:/etc/kubernetes/
```

`复制加入master join命令在 master-2 执行`

```bash
kubeadm join 192.168.148.73:16443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:78c0233ca7d519a9faab74bf55ce194eb9aba2dcbcb578d29614b979421e7424 \
	--control-plane
```

`拷贝kubectl使用的连接k8s认证文件到 master-2 默认路径`

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
NAME       STATUS     ROLES           AGE     VERSION
master-1   NotReady   control-plane   6m23s   v1.25.0
master-2   NotReady   control-plane   39s     v1.25.0
```

`访问负载均衡器测试`

```bash
# 任意节点执行
curl -k https://192.168.148.73:16443/version
{
  "major": "1",
  "minor": "25",
  "gitVersion": "v1.25.0",
  "gitCommit": "a866cbe2e5bbaa01cfd5e969aa3e033f3282a8a2",
  "gitTreeState": "clean",
  "buildDate": "2022-08-23T17:38:15Z",
  "goVersion": "go1.19",
  "compiler": "gc",
  "platform": "linux/amd64"
}
# curl -> vip(nginx) -> apiserver
# 查看Nginx日志也可以看到转发apiserver IP
tail /var/log/nginx/k8s-access.log -f
192.168.148.71 192.168.148.71:6443, 192.168.148.70:6443 - [13/Apr/2023:04:13:26 -0400] 200 0, 667
192.168.148.71 192.168.148.70:6443 - [13/Apr/2023:04:13:26 -0400] 200 977
192.168.148.71 192.168.148.70:6443 - [13/Apr/2023:04:13:26 -0400] 200 2224
192.168.148.71 192.168.148.71:6443 - [13/Apr/2023:04:14:04 -0400] 200 3640
192.168.148.71 192.168.148.70:6443 - [13/Apr/2023:04:14:42 -0400] 200 1863
192.168.148.72 192.168.148.70:6443 - [13/Apr/2023:04:15:27 -0400] 200 428
```

## 加入Kubernetes Node

`在 master-1 上查看加入节点的命令`

```bash
kubeadm token create --print-join-command
kubeadm join 192.168.148.73:16443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:78c0233ca7d519a9faab74bf55ce194eb9aba2dcbcb578d29614b979421e7424
```

`把 node-1 加入k8s集群`
在 node1 执行

```bash
kubeadm join 192.168.148.73:16443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:78c0233ca7d519a9faab74bf55ce194eb9aba2dcbcb578d29614b979421e7424
```

`默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：`

```bash
kubeadm token create
kubeadm token list
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
63bca849e0e01691ae14eab449570284f0c3ddeea590f8da988c07fe2729e924

kubeadm join 192.168.148.70:6443 --token nuja6n.o3jrhsffiqs9swnu --discovery-token-ca-cert-hash sha256:63bca849e0e01691ae14eab449570284f0c3ddeea590f8da988c07fe2729e924
```

kubeadm token create --print-join-command

<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/>

## 网络方案（CNI）

 <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network> 

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
kubectl get node
NAME       STATUS   ROLES           AGE   VERSION
master-1   Ready    control-plane   23m   v1.25.0
master-2   Ready    control-plane   17m   v1.25.0
node-1     Ready    <none>          11m   v1.25.0
```

`测试在k8s创建pod是否可以正常访问网络`

```bash
kubectl run busybox --image busybox:1.28 --image-pull-policy=IfNotPresent --restart=Never --rm -it busybox -- sh
 # 通过下面可以看到能访问网络，说明calico网络插件已经被正常安装了
/ # ping www.baidu.com
PING www.baidu.com (14.119.104.254): 56 data bytes
64 bytes from 14.119.104.254: seq=0 ttl=127 time=27.637 ms
...
```

`测试coredns是否正常`

```bash
kubectl run busybox --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # nslookup kubernetes.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default.svc.cluster.local
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
```