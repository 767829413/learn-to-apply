# 在VMWare中部署你的K8S集群-二进制

## 安装要求

基础的虚拟机配置可以参考:

[在VMWare中部署你的K8S集群-kubeadm](../../docs/Kubernetes/quick-start-k8s-cluster-kubeadm.md)

基本配置:

```bash
[root@master ~]# cat /etc/redhat-release
CentOS Linux release 7.6.1810 (Core)
[root@master ~]# uname -a
Linux master 3.10.0-957.el7.x86_64 #1 SMP Thu Nov 8 23:39:32 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```

| 角色       | IP            |
| ---------- | ------------- |
| master | 192.168.148.65 |
| node-1  | 192.168.148.66 |
| node-2  | 192.168.148.67 |
| gateway  | 192.168.148.2 |

## 创建CA根证书和秘钥

### 安装cfssl工具集

[cfssl, cfssljson, cfssl-certinfo](https://github.com/cloudflare/cfssl/releases)

```bash
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssl-certinfo_1.6.3_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssljson_1.6.3_linux_amd64
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.3/cfssl_1.6.3_linux_amd64
mkdir ~/cfssl && cd ~/cfssl
mv ../cfssl-certinfo_1.6.3_linux_amd64 ./cfssl-certinfo
mv ../cfssl_1.6.3_linux_amd64 ./cfssl
mv ../cfssljson_1.6.3_linux_amd64 ./cfssljson
```

### 创建根证书(CA)

#### 创建配置文件

```bash
mkdir -p ~/k8s/ca && cd ~/k8s/ca
# signing：表示该证书可用于签名其它证书（生成的 ca.pem 证书中 CA=TRUE）
# server auth：表示 client 可以用该该证书对 server 提供的证书进行验证
# client auth：表示 server 可以用该该证书对 client 提供的证书进行验证
# "expiry": "876000h"：证书有效期设置为 100 年
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
EOF
```

#### 创建证书签名请求文件

```bash
# CN 用户名
# O 用户组
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes-ca",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "k8s",
      "OU": "dqz"
    }
  ],
  "ca": {
    "expiry": "876000h"
 }
}
EOF
```

#### 生成CA证书和私钥

```bash
~/cfssl/cfssl gencert -initca ca-csr.json | ~/cfssl/cfssljson -bare ca
2023/03/31 02:40:56 [INFO] generating a new CA key and certificate from CSR
2023/03/31 02:40:56 [INFO] generate received request
2023/03/31 02:40:56 [INFO] received CSR
2023/03/31 02:40:56 [INFO] generating key: rsa-2048
2023/03/31 02:40:56 [INFO] encoded CSR
2023/03/31 02:40:56 [INFO] signed certificate with serial number 551442106332540930413295942991634393679848928555
ls -l
total 20
-rw-r--r-- 1 root root  293 Mar 31 02:35 ca-config.json
-rw-r--r-- 1 root root 1045 Mar 31 02:40 ca.csr
-rw-r--r-- 1 root root  246 Mar 31 02:39 ca-csr.json
-rw------- 1 root root 1679 Mar 31 02:40 ca-key.pem
-rw-r--r-- 1 root root 1314 Mar 31 02:40 ca.pem
```

#### 分发证书文件

```bash
mkdir -p  /etc/kubernetes/cert && cp ca*.pem /etc/kubernetes/cert
ssh root@192.168.148.66 "mkdir -p /etc/kubernetes/cert"
scp ca*.pem ca-config.json root@192.168.148.66:/etc/kubernetes/cert
ssh root@192.168.148.67 "mkdir -p /etc/kubernetes/cert"
scp ca*.pem ca-config.json root@192.168.148.67:/etc/kubernetes/cert
```

## 部署ETCD集群

etcd 是基于 Raft 的分布式 KV 存储系统，由 CoreOS 开发，常用于服务发现、共享配置以及并发控制（如 leader 选举、分布式锁等）。
kubernetes 使用 etcd 集群持久化存储所有 API 对象、运行数据。
etcd 集群节点名称和 IP 如下：
master：192.168.148.65

### 下载和安装

[ETCD仓库地址](https://github.com/etcd-io/etcd/releases)

```bash
mkdir ~/k8s/work/etcd && cd ~/k8s/work/etcd
```

创建一个install.sh文件如下

```bash
#!/bin/bash

ETCD_VER=v3.4.24

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C ~/k8s/work/etcd --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

~/k8s/work/etcd/etcd --version
~/k8s/work/etcd/etcdctl version
```

```bash
bash install.sh
etcd Version: 3.4.24
Git SHA: 6d1bfe4f9
Go Version: go1.17.13
Go OS/Arch: linux/amd64
etcdctl version: 3.4.24
API version: 3.4
```

### 创建 etcd 证书和私钥

#### 创建证书签名请求

`这里的 hosts: IP地址一定要根据自己的实际ETCD集群IP填写`

```bash
mkdir -p ~/k8s/work/etcd/cert/ && cd ~/k8s/work/etcd/cert/
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.148.65"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "k8s",
      "OU": "dqz"
    }
  ]
}
EOF
```

#### 生成证书和私钥

```bash
~/cfssl/cfssl gencert -ca=/root/k8s/ca/ca.pem \
    -ca-key=/root/k8s/ca/ca-key.pem \
    -config=/root/k8s/ca/ca-config.json \
-profile=kubernetes etcd-csr.json | ~/cfssl/cfssljson -bare etcd
2023/03/31 05:10:27 [INFO] generate received request
2023/03/31 05:10:27 [INFO] received CSR
2023/03/31 05:10:27 [INFO] generating key: rsa-2048
2023/03/31 05:10:27 [INFO] encoded CSR
2023/03/31 05:10:27 [INFO] signed certificate with serial number 69256132031264234236833124182491128097946099019
```

### 创建 etcd 的 systemd 配置文件

参考:<https://etcd.io/docs/v3.2/platforms/container-linux-systemd/>

注意填写:

```text
ExecStart={your_path}/etcd
  --cert-file={your_path}/etcd.pem \\
  --key-file={your_path}/etcd-key.pem \\
  --trusted-ca-file={your_path}/ca.pem \\
  --peer-cert-file={your_path}/etcd.pem \\
  --peer-key-file={your_path}/etcd-key.pem \\
  --peer-trusted-ca-file={your_path}/ca.pem \\
  --listen-peer-urls=https:/{your_ip}:2380 \\
  --initial-advertise-peer-urls=https:/{your_ip}:2380 \\
  --listen-client-urls=https:/{your_ip}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https:/{your_ip}:2379 \\
  --initial-cluster=etcd-192.168.148.65-service=https:/{your_ip}:2380
```

```bash
mkdir -p /root/k8s/work/service-conf/etcd && cd /root/k8s/work/service-conf/etcd
cat > etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/root/k8s/work/etcd/data
ExecStart=/root/k8s/work/etcd/etcd \\
  --data-dir=/root/k8s/work/etcd/data \\
  --wal-dir=/root/k8s/work/etcd/wal \\
  --name=etcd-192.168.148.65-service \\
  --cert-file=/root/k8s/work/etcd/cert/etcd.pem \\
  --key-file=/root/k8s/work/etcd/cert/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-cert-file=/root/k8s/work/etcd/cert/etcd.pem \\
  --peer-key-file=/root/k8s/work/etcd/cert/etcd-key.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls=https://192.168.148.65:2380 \\
  --initial-advertise-peer-urls=https://192.168.148.65:2380 \\
  --listen-client-urls=https://192.168.148.65:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://192.168.148.65:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=etcd-192.168.148.65-service=https://192.168.148.65:2380 \\
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
cp ./etcd.service /etc/systemd/system/etcd.service
```

### 启动ETCD服务

* 必须创建 etcd 数据目录和工作目录;
* 注意：3.4.10+版本，需要将数据目录的权限设置为0700才可以正常启动

```bash
export ETCD_WAL_DIR=/root/k8s/work/etcd/wal
export ETCD_DATA_DIR=/root/k8s/work/etcd/data
mkdir -p ${ETCD_DATA_DIR} ${ETCD_WAL_DIR} && chmod 0700 ${ETCD_DATA_DIR}
systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd
systemctl status etcd.service
● etcd.service - Etcd Server
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-03-31 06:31:23 EDT; 11s ago
     Docs: https://github.com/coreos
 Main PID: 28661 (etcd)
   CGroup: /system.slice/etcd.service
           └─28661 /root/k8s/work/etcd/etcd --data-dir=/root/k8s/work/etcd/data --wal-dir=/root/k8s/work/etcd/wal --name=etcd-192.168.148.65-service --cert-file=/root/k8s/work/etcd/cert/etcd.pem --key-file=/root/k8s/work/etcd/cert/etcd-key.pem --trus...

Mar 31 06:31:23 master etcd[28661]: published {Name:etcd-192.168.148.65-service ClientURLs:[https://192.168.148.65:2379]} to cluster 63f7791745e61050
Mar 31 06:31:23 master etcd[28661]: raft2023/03/31 06:31:23 INFO: raft.node: b9916c2697476722 elected leader b9916c2697476722 at term 2
Mar 31 06:31:23 master etcd[28661]: ready to serve client requests
Mar 31 06:31:23 master etcd[28661]: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
Mar 31 06:31:23 master etcd[28661]: setting up the initial cluster version to 3.4
Mar 31 06:31:23 master etcd[28661]: ready to serve client requests
Mar 31 06:31:23 master etcd[28661]: set the initial cluster version to 3.4
Mar 31 06:31:23 master etcd[28661]: enabled capabilities for version 3.4
Mar 31 06:31:23 master etcd[28661]: serving client requests on 192.168.148.65:2379
Mar 31 06:31:23 master systemd[1]: Started Etcd Server.
```

### 验证服务状态

```bash
/root/k8s/work/etcd/etcdctl \
    --endpoints=https://192.168.148.65:2379 \
    --cacert=/etc/kubernetes/cert/ca.pem \
    --cert=/root/k8s/work/etcd/cert/etcd.pem \
    --key=/root/k8s/work/etcd/cert/etcd-key.pem endpoint health
https://192.168.148.65:2379 is healthy: successfully committed proposal: took = 5.097402ms
```
