# 搭建dapr基于k8s的开发调试环境

## 前置条件

* [使用win10的wsl2](../../docs/Productivity/wsl2-dev.md)
* golang版本1.17及以上版本
* IDE建议vs code

### 安装kind

<https://kind.sigs.k8s.io/docs/user/quick-start/>

### 安装kubectl

<https://kubernetes.io/docs/tasks/tools/>

### 基于Kind创建

<https://docs.dapr.io/operations/hosting/kubernetes/cluster/setup-kind/>

### 安装Helm

<https://helm.sh/docs/intro/install/>

### 基于k8s进行调试

<https://docs.dapr.io/developing-applications/debugging/debug-k8s/debug-dapr-services/>

#### 构建本地k8s测试环境 后缀参数DEBUG=1

* 清理旧的dapr k8s集群部署

```bash
dapr uninstall -k
```

* 环境变量配置(Linux/macOS)

```bash
export DAPR_REGISTRY=docker.io/<your_docker_hub_account>
export DAPR_TAG=dev
```

* 本地docker镜像执行并推送(Kind集群可以不推送)

```bash
make docker-build DEBUG=1
make docker-push DEBUG=1
```

* 将镜像加载入Kind进群环境(推送到dockerhub可以忽略这步)

```bash
make docker-push-kind DEBUG=1
```

* 将本地镜像部署到k8s环境

```bash
make docker-deploy-k8s DEBUG=1
```

* 获取pod数目,进行端口暴露

```bash
kubectl get pods -n dapr-system -o wide
kubectl port-forward dapr-operator-xx-xx 40000:40000 -n dapr-system
```
