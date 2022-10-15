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
nohup docsify serve /your_path/docs --port 80 > ./info.log 2>&1& echo $! > ./info.pid
```

这里其实建议配合nginx来使用,还能兼顾https,可以参考我的这篇文章

[通过Docker快速搭建一个静态资源服务器](../../docs/Play/static_resource_server.md)

### 步骤四 利用github的webhook进行自动更新

这里使用的是github的[webhook](https://docs.github.com/cn/developers/webhooks-and-events/webhooks/about-webhooks)来实现自动拉取的功能,方式有很多,自己google就是了,这里也提供了一个简单的执行方案[webhook_linux_amd64](https://github.com/767829413/webhook)

```bash
## route 定义在webhook配置的Payload URL
## path 项目路径执行git pull的地方
## port 监听的端口
## secret 定义在webhook配置的secret
nohup ./webhook_linux_amd64 -route=your_route -path=your_path -port=your_port -secret=your_secret > ./webhook.log 2>&1& echo $! > ./webhook.pid
```
