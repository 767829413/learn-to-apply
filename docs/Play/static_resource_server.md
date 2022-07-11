# 通过Docker快速搭建一个nginx

## 在云服务器上快速配置nginx

### 前提条件

* 买一个域名
* 拥有自己的服务器(本例用的是阿里云轻量应用服务器)
* 把你的域名DNS解析至你的云公网IP(生效时间预计从0-48小时不等)
* 了解一下[Docker基本操作](https://yeasy.gitbook.io/docker_practice/)

### 步骤一 构建Docker镜像(基于Nginx镜像)

**新建一个mynginx工程来构建Docker镜像(基于Nginx镜像)**

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

这里给出一份nginx配置

```conf
user root;
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
  worker_connections 1024;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  log_format main '$remote_addr - $remote_user [$time_local] "$request" '
  '$status $body_bytes_sent "$http_referer" '
  '"$http_user_agent" "$http_x_forwarded_for"';

  access_log /var/log/nginx/access.log main;

  sendfile on;
  #tcp_nopush     on;

  keepalive_timeout 65;

  gzip on;

  #include /etc/nginx/conf.d/*.conf;
  server {
    listen 80;
    server_name domain1 domain2;

    location / {
      root /var/www; 
      index index.html index.htm;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
      root /usr/share/nginx/html;
    }
  }
}
```

### 步骤二 构建和启动Docker镜像和容器

* 构建镜像

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
