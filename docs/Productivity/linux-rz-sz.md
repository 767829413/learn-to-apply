# LINUX上传下载命令：RZ,SZ

## 简介

有时候在操作xshell需要上传本地文件的时候还需要打开ftp来上传,我感觉有点麻烦,Linux中的sz和rz就可以完美解决这个问题.
sz即使send Zmodem,就是用Zmodem文件传输协议从Linux服务器发送文件到window的意思,rz则就是receive Zmodem,从字面就很容易理解是在Linux上接收文件,也就是上传了
rz与sz需要端支持,终端就是连接远程服务器的客户端.例如 XShell、SecureCRT 等,linux默认终端是不支持的.

### 安装

```bash
sudo apt-get -y install lrzsz
```

### 使用

```bash
# sz filename[你的文件名] 弹窗让你选择要下载存放的目录
# rz 弹窗选择文件(多选支持批量上传)
```
