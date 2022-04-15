# 通过Docker快速搭建一个静态资源服务器

## 在云服务器上搭建搭建一个静态资源服务器

### 前提条件

* 买一个域名
* 拥有自己的服务器(本例用的是阿里云轻量应用服务器)
* 把你的域名DNS解析至你的云公网IP（生效时间预计从0-48小时不等）
* 了解一下[Docker基本操作](https://yeasy.gitbook.io/docker_practice/)

### 步骤一 构建Docker镜像（基于Nginx镜像）

**新建一个hexo-docker工程来构建Docker镜像（基于Nginx镜像），里面有且仅包含Dockerfile和初始Nginx配置。Dockerfile中描述了Githooks挂载地址，安装好了Certbot命令，并加入初始Nginx配置**

```Dockerfile
#基于Nginx镜像
FROM nginx:latest

#换国内的源，这里用网易源
RUN sed -i 's#http://deb.debian.org#https://mirrors.163.com#g' /etc/apt/sources.list && sed -i 's#http://security.debian.org#https://mirrors.163.com#g' /etc/apt/sources.list && rm -Rf /var/lib/apt/lists/* && apt-get update

#SSH初始化
RUN apt-get install -y openssh-server && ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key -y && ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key -y && ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key -y

#安装Certbot
RUN apt-get install -y certbot && apt-get install -y python3-certbot-nginx

#暴露22 SSH
#暴露80 HTTP
#暴露443 HTTPS
EXPOSE 22
EXPOSE 80
EXPOSE 443
```

### 步骤二 构建和启动Docker镜像和容器

* 构建镜像

```bash
docker build -t hexo-docker .
```

* 启动容器

    ***这里是通过volume方案，为后续申请证书的挂载做准备***

```bash
docker run --name nginx -v $(pwd)/letsencrypt:/etc/letsencrypt -v /var/www/blog:/var/www/blog -v $(pwd)/nginx/nginx.conf:/etc/nginx/nginx.conf -d -p 80:80 -p 443:443 -p 8004:22 hexo-docker
```

* 配置证书

  * 进入容器

    ```bash
    docker exec -it nginx /bin/bash
    ```

  * 申请证书

    ***我们在用Dockfile来构建的时候已经安装了Certbot，我们可以利用Certbot帮忙安装Https证书，同时Certbot会自动识别上文中配置的Nginx文件，加入证书验证和443端口开放***

    ```bash
    ## 申请的域名
    certbot --nginx -d "xxx.xxx"
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
