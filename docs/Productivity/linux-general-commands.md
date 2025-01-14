# LINUX 常用命令

## 定时输出实时查看-n 1 选项设置更新间隔为1秒

```bash
watch -n 1 -t ls -lh /tmp
```

## snap 更新

```bash
sudo snap refresh
```

## snap-store 更新

```bash
sudo snap refresh snap-store
snap-store --quit-on-close-for-update
```

## 解压缩

```bash
tar -xf archive.tar.xz
```

## 日志查看

```bash
# 基本
journalctl -n 50
journalctl --since "1 hour ago"
journalctl --since "<start_time>" --until "<end_time>"
# 获取从昨天到现在的日志
journalctl --since "yesterday" --until "now"
# 获取 nginx 服务的最新 100 行日志
journalctl -u nginx.service -n 100
# 获取错误级别及以上的最新 50 行日志
journalctl -p err -n 50
```

# dns服务

```bash
sudo systemctl start systemd-resolved
sudo resolvectl flush-caches
sudo systemctl restart systemd-resolved
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

# 代理

```bash
export http_proxy='http://127.0.0.1:7897'
export https_proxy='http://127.0.0.1:7897'

[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7897/"
Environment="HTTPS_PROXY=http://127.0.0.1:7897/"
```

# pip 源头

```text
-i http://mirrors.tencent.com/pypi/simple --trusted-host mirrors.tencent.com
```

## rz sz

有时候在操作xshell需要上传本地文件的时候还需要打开ftp来上传,我感觉有点麻烦,Linux中的sz和rz就可以完美解决这个问题.
sz即使send Zmodem,就是用Zmodem文件传输协议从Linux服务器发送文件到window的意思,rz则就是receive Zmodem,从字面就很容易理解是在Linux上接收文件,也就是上传了
rz与sz需要端支持,终端就是连接远程服务器的客户端.例如 XShell、SecureCRT 等,linux默认终端是不支持的.

```bash
sudo apt-get -y install lrzsz
```

```bash
# sz filename[你的文件名] 弹窗让你选择要下载存放的目录
# rz 弹窗选择文件(多选支持批量上传)
```

## echo 文字到文本

* shell 脚本 echo 单行文字到文本

    ```bash
    #!/bin/bash
    echo "test测试" > a.txt
    ```

* shell 脚本 echo 多行文字到文本

    ```bash
    #!/bin/bash
    cat >a.txt<<EOF
    test测试
    多行文本
    重定向
    到文件
    EOF
    ```

    同下

    ```bash
    #!/bin/bash
    cat >a.txt<<ABC
    test测试
    多行文本
    ABC
    ```

* Dockerfile 中 echo 多行文字到文本

    ```Dockerfile
    FROM ubuntu:18.04

    RUN mv /etc/apt/sources.list /etc/apt/sources_bak.list \
    && echo 'deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse\n\
    deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse\n\
    \n\
    deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse\n\
    deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse\n\
    \n\
    deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse\n\
    deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse\n\
    \n\
    deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse\n\
    deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse\n\
    \n\
    deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse\n\
    deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse\n'\
    > /etc/apt/sources.list
    ```

## sed

[详细说明](https://wangchujiang.com/linux-command/c/sed.html)

```bash
# -n选项 和 p命令 一起使用表示只打印那些发生替换的行
sed -n 's/test/TEST/p' file
# 直接编辑文件 选项-i ，会匹配file文件中每一行的所有book替换为books
sed -i 's/book/books/g' file
# 使用后缀 /g 标记会替换每一行中的所有匹配
sed 's/book/books/g' file
```

## shell 输出输入

```bash
#!/bin/bash

EXE=./local
LOG=./local.log
PID=./local.pid

if [ ! -f "$PID" ]; then
    echo "$PID not exist,need create"
    touch $PID
fi

cat $PID | xargs kill
chmod +x $EXE
nohup $EXE > $LOG 2>&1& echo $! > $PID
```