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

replicasets 副本集,帮助 deployment 完成副本数管理,回滚

pod 最小部署单元,容器的更高级抽象

service 应用暴露

2. 发布

```bash
kubectl expose deployment demo1 --port=80 --target-port=8080 --name=demo1 --type=NodePort

kubectl get svc
```

## 3. 服务编排（YAML）

yaml 是一种简洁的非标记语言

语法格式:

* 缩进表示层级关系
* 不支持制表符 "tab" 缩进,使用空格缩进
* 通常开头缩进2空格
* 字符后面缩进1个空格,如冒号,逗号等
* "---" 表示YAML格式,一个文件的开始
* "#" 开头表示注释

```yml
apiVersion: apps/v1 # 控制器定义
kind: Deployment
metadata:
  name: demo2
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo2
  template: # 被控制对象
    metadata:
      labels:
        app: demo2
    spec:
      containers:
        - name: web
          image: lizhenliang/java-demo
          ports:
            - containerPort: 80

---
# 暴露服务的service
apiVersion: v1
kind: Service
metadata:
  name: demo2
  namespace: default
spec:
  ports:
  - nodePort: 30219
    port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: demo2
  type: NodePort
```

| 名称 | 解释 |
| --- | --- |
| apiVersion | API版本 |
| kind | 资源类型 |
| metadata | 资源元数据 |
| spec | 资源规格 |
| replicas | 副本数量 |
| selector | 标签选择器 |
| template | Pod模板 |
| metadata | Pod元数据 |
| spec | Pod规格 |
| containers | 容器配置 |

```bash
kunectl apply -f ./demo2.yml
```

命令行与yaml的优缺点:

* 命令行适合快速部署
* yaml通过文件描叙资源,方便服用,管理容易

利用命令行生成yaml模板

```bash
# create命令生成
kubectl create deployment demo3 --image=lizhenliang/java-demo --dry-run=client -o yaml > demo3.yaml
# get命令导出
kubectl get svc demo2 -o yaml > demo2_svc.yaml
# 使用explain字段输出
kubectl explain  pods.spec
```

## 4. 应用升级、弹性伸缩、回滚、删除

镜像是容器化的交付物,镜像有自己的版本管理,可以通过镜像版本作为项目的版本

应用升级

```bash
kubectl set image deployment web nginx=nginx:1.17 [--record=true]
# 查看升级或回滚状态
kubectl rollout status deployment web
```

应用回滚

```bash
# 查看版本历史
kubectl rollout history deployment web
# 回滚到上一版本
kubectl rollout undo deployment web
# 指定回滚版本
kubectl rollout undo deployment web --to-revision=1
# 查看升级或回滚状态
kubectl rollout status deployment web
```

弹性伸缩

```bash
# 扩容 | 缩容 设置--replicas
kubectl scale deployment web --replicas=10
# 自动扩容 | 缩容
kubectl autoscale deployment web --max=3 --min=1 --cpu-percent=30
```

删除

```bash
kubectl delete deployment web
kubectl delete service web
```

## 5. Pod对象：Pod设计思想、应用自修复、Init container 、静态Pod等

### Pod对象: 基本概念

* 最小部署单元
* 一组容器集合
* 一个pod中的容器共享网络命名空间
* Pod是暂时的

### Pod对象: 存在意义

[关于对 Kubernetes 原子调度单位Pod的思考](../../docs/Kubernetes/k8s-atomic-Scheduling.md)

* Infrastructure Container: 基础容器
  * 维护整个Pod网络空间
* InitContainers:初始化容器
  * 先于业务容器开始执行
* Containers:业务容器
  * 并行启动

pod 解决问题:

1. 容器之间的网络,先创建一个infrastructure container,后面的业务容器加入到这个infra container的网络命名空间,这样业务容器之间就网络共享
2. 容器之间的文件利用数据卷来实现文件共享,访问

Pod为亲密性应用存在

亲密性应用场景:

* 两个应用之间发生文件交互
* 两个应用需要通过127.0.0.1或者socket通信(nginx + php)
* 两个应用发生频繁调用

### Pod对象: 应用自恢复

[配置教程](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo3
  name: demo3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo3
  strategy: {}
  template:
    metadata:
      labels:
        app: demo3
    spec:
      restartPolicy: Always
      containers:
        - image: lizhenliang/java-demo
          name: java-demo
          livenessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 20
          readinessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 30 # 延迟多少时间后执行
            periodSeconds: 20 # 每隔一段时间执行一次检查
```

重启策略(restartPolicy)：

* Always：当容器终止退出后，总是重启容器，默认策略。
* OnFailure：当容器异常退出（退出状态码非0）时，才重启容器。
* Never：当容器终止退出，从不重启容器。

健康检查有以下两种类型：

* livenessProbe（存活检查）
如果检查失败，将杀死容器，根据Pod的restartPolicy来操作。
* readinessProbe（就绪检查）
如果检查失败，Kubernetes会把Pod从service endpoints中剔除。

支持以下三种检查方法：

* httpGet：发送HTTP请求，返回200-400范围状态码为成功。
* exec：执行Shell命令返回状态码是0为成功。
* tcpSocket：发起TCP Socket建立成功

### Pod对象: Init Container

```yml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
      volumeMounts:
        - name: workdir
          mountPath: /usr/share/nginx/html
  initContainers:
    - name: install
      image: busybox
      command:
        - wget
        - "-O"
        - "/work-dir/index.html"
        - http://kubernetes.io
      volumeMounts:
        - name: workdir
          mountPath: "/work-dir"
  dnsPolicy: Default
  volumes:
    - name: workdir
      emptyDir: {}
```

`Init container：`

* 基本支持所有普通容器特征
* 优先普通容器执行

应用场景：

* 控制普通容器启动
* 初始化配置

### Pod对象: 静态Pod

静态Pod：固定在某个Node上面，由kubelet管理生成的一种Pod。

* 有特定节点上的kubelet管理
* 没有任何控制器
* pod挂了,kubelet去自动拉起
* kubelet定期检查目录 /etc/kubernetes/manifests/ 创建或删除
