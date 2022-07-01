# 深入理解声明式API-实践自定义控制器

## 利用kuberbuilder快速实践CRD与自定义控制器

### 前提条件

* [kubebuilder安装](https://book.kubebuilder.io/quick-start.html)
* [本地WSL环境搭建](../../docs/Productivity/wsl2-dev.md)
* [安装kubectl](https://kubernetes.io/docs/tasks/tools/)
* [安装kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

我本地的 `golang` `docker` `kubernetes` 版本

```bash
go version
# go version go1.18.2 linux/amd64
kubectl version --output=yaml
# clientVersion:
#   buildDate: "2022-05-03T13:46:05Z"
#   compiler: gc
#   gitCommit: 4ce5a8954017644c5420bae81d72b09b735c21f0
#   gitTreeState: clean
#   gitVersion: v1.24.0
#   goVersion: go1.18.1
#   major: "1"
#   minor: "24"
#   platform: linux/amd64
# kustomizeVersion: v4.5.4
# serverVersion:
#   buildDate: "2022-05-19T15:39:43Z"
#   compiler: gc
#   gitCommit: 4ce5a8954017644c5420bae81d72b09b735c21f0
#   gitTreeState: clean
#   gitVersion: v1.24.0
#   goVersion: go1.18.1
#   major: "1"
#   minor: "24"
#   platform: linux/amd64
docker version
# Client: Docker Engine - Community
#  Cloud integration: v1.0.25
#  Version:           20.10.16
#  API version:       1.41
#  Go version:        go1.17.10
#  Git commit:        aa7e414
#  Built:             Thu May 12 09:17:39 2022
#  OS/Arch:           linux/amd64
#  Context:           default
#  Experimental:      true
# 
# Server: Docker Desktop
#  Engine:
#   Version:          20.10.16
#   API version:      1.41 (minimum version 1.12)
#   Go version:       go1.17.10
#   Git commit:       f756502
#   Built:            Thu May 12 09:15:42 2022
#   OS/Arch:          linux/amd64
#   Experimental:     false
#  containerd:
#   Version:          1.6.4
#   GitCommit:        212e8b6fa2f44b9c21b2798135fc6fb7c53efc16
#  runc:
#   Version:          1.1.1
#   GitCommit:        v1.1.1-0-g52de29d
#  docker-init:
#   Version:          0.19.0
#   GitCommit:        de40ad0
```

### 步骤一 kubebuilder 初始化操作以及相应Api创建

在自己的GOPATH下面创建一个相面文件夹,相关操作查阅[kubebuilder快速开始文档](https://book.kubebuilder.io/quick-start.html)

```bash
# 项目文件夹创建
mkdir -p $GOPATH/src/kubebuilder-demo
cd $GOPATH/src/kubebuilder-demo

# 使用kubebuilder初始化,等待完成
kubebuilder init --domain demo.kubebuilder.io

# 创建API,根据提示操作
kubebuilder create api --group myapp --version v1 --kind Redis

# 当前项目层级
tree -L 1
# .
# ├── api
# ├── bin
# ├── config
# ├── controllers
# ├── Dockerfile
# ├── go.mod
# ├── go.sum
# ├── hack
# ├── main.go
# ├── Makefile
# ├── PROJECT
# └── README.md
# 
# 5 directories, 7 files

```

### 步骤二 demo开发实践

主要是为了创建自定义 [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 实现自定义的 `Redis` 类型资源相关 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) 的功能:

* 定义 `CRD` 规格
* 创建 `CRD` 对应的资源 `Redis`
* 实现资源 `Redis` 创建相应 `Pod`,支持 `副本数,验证,扩容收缩`

```bash
docker build -t mynginx .
```

* 启动容器

    ***这里是通过volume方案，为后续申请证书的挂载做准备***

```bash
docker run --name nginx -v /your_path/letsencrypt:/etc/letsencrypt -v /your_path/var/www:/var/www -v /your_path/conf/nginx.conf:/etc/nginx/nginx.conf -d -p 80:80 -p 443:443 -p 8004:22 mynginx
```

* 配置证书

  * 进入容器

    ```bash
    docker exec -it mynginx /bin/bash
    ```

  * 申请证书

    ***我们在用Dockfile来构建的时候已经安装了Certbot，我们可以利用Certbot帮忙安装Https证书，同时Certbot会自动识别上文中配置的Nginx文件，加入证书验证和443端口开放***

    ```bash
    ## 申请的域名
    certbot --nginx -d "domain1","domain2"
    ```

    ***这里就按照提示一步一步走就是了，最后见到congratulation!就能证明证书已经安装好了，因为我们在docker run 时采用了volume方案，生成出的证书会挂载到云服务的$(pwd)/letsencrypt目录下，并关联到Docker内部的/etc/letsencrypt目录 。这样能保证镜像中不带CA证书，解耦的同时也确保Docker在迁移的时候的安全性。***

  * 证书更新

    ```bash
    /usr/bin/certbot renew
    ```

    ***Letsencrypt 3个月就需要更新一次，很多人用自动更新任务来完成更新。我这边就不想用了，因为我觉得每个月收到提示邮件后上云更新一把是一件很愉快的事。如果你需要自动更新可以使用下面命令***

    ```bash
    echo "0 0 1 * * /usr/bin/certbot renew --quiet" >> /etc/crontabs/root
    ```
