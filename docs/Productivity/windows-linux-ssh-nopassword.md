# 在win10和Linux上配置SSH免密登录

## 服务器SSH免密登录

### 产生SSH公私钥的方法

```bash
ssh-keygen -t rsa -b 4096 -C "xxxxxx@example.com"
```

### 向服务器上传公钥

在本地Linux (或macOS) 上使用 ssh-copy-id 的方式上传公钥

```bash
ssh-copy-id -i id_rsa.pub 用户名@IP -p 22
```

在windows上使用 ssh-copy-id 的方式
利用wsl

```bash
cd /mnt/c/Users/xx/.ssh
ssh-copy-id -i id_rsa.pub 用户名@IP -p 22
```
