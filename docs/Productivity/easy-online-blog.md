# 搭建自己的简易Blog

## 在服务器上快速搭建docsify

### 前提条件

* 拥有自己的服务器(本例是ubuntu 20.04)
* 稍微看下[docsify的快速开始](https://docsify.js.org/#/quickstart)

### 步骤一 通过Ubuntu20.04软件源安装 Node.js 和 npm

包含在 Ubuntu 20.04 软件源中的 Node.js 版本是10.19.0,这是一个长期版本,运行下面的命令更新软件包索引，并且安装 Node.js 和 npm:

```bash
sudo apt-get update
sudo apt-get -y install nodejs npm
#查看版本nodejs
nodejs --version
#查看npm版本
npm --version
```

### 步骤二 通过 npm 安装 docsify-cli

```bash
npm i docsify-cli -g
```

### 步骤三 利用docsify快速搭建

```bash
## 初始化文档地址
docsify init /your_path/docs
## 后台运行
nohup docsify serve ./learn-to-apply --port 80 > ./info.log 2>&1& echo $! > ./info.pid
```

### 步骤四 利用github的webhook进行自动更新
