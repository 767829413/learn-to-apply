# 应用生命周期管理

## 1. 在Kubernetes中部署应用流程

* 制作镜像: dockerfile
    1. 定制化镜像
    2. 标准化

* 控制器管理Pod

* 暴露应用

* 对外发布应用

* 日志/监控

## 2. 使用Deployment部署应用

使用命令行快速部署

1. 创建

```bash
kubectl create deployment web --image=lizhenliang/java-demo
kubectl get deploy 
kubectl get pods
```

deployment 部署控制器,主要部署像api,网站,微服务

replicasets 副本集,帮助 

deployment 完成副本数管理,回滚

pod 最小部署单元,容器的更高级抽象

2. 发布

```bash
kubectl expose deployment demo1 --port=80 --target-port=8080 --name=demo1 --type=NodePort

kubectl get svc
```

## 3. 服务编排（YAML）

## 4. 应用升级、弹性伸缩、回滚、删除

## 5. Pod对象：Pod设计思想、应用自修复、Init container 、静态Pod等
