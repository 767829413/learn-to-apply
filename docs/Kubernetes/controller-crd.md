# 深入理解声明式API-实践自定义控制器

## 利用kuberbuilder快速实践CRD与自定义控制器

### 前提条件

* [kubebuilder安装](https://book.kubebuilder.io/quick-start.html)
* [本地WSL环境搭建](../../docs/Productivity/wsl2-dev.md)
* [安装kubectl](https://kubernetes.io/docs/tasks/tools/)
* [安装kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
* [Kubectl 命令表](http://docs.kubernetes.org.cn/683.html)

我本地的 `golang` `docker` `kubernetes` 版本

```bash
go version
# go version go1.18.2 linux/amd64
```

```bash
kubectl version --output=yaml
```

```yaml
clientVersion:
  buildDate: "2022-05-03T13:46:05Z"
  compiler: gc
  gitCommit: 4ce5a8954017644c5420bae81d72b09b735c21f0
  gitTreeState: clean
  gitVersion: v1.24.0
  goVersion: go1.18.1
  major: "1"
  minor: "24"
  platform: linux/amd64
kustomizeVersion: v4.5.4
serverVersion:
  buildDate: "2022-05-19T15:39:43Z"
  compiler: gc
  gitCommit: 4ce5a8954017644c5420bae81d72b09b735c21f0
  gitTreeState: clean
  gitVersion: v1.24.0
  goVersion: go1.18.1
  major: "1"
  minor: "24"
  platform: linux/amd64
```

```bash
docker version
```

```text
Client: Docker Engine - Community
 Cloud integration: v1.0.25
 Version:           20.10.16
 API version:       1.41
 Go version:        go1.17.10
 Git commit:        aa7e414
 Built:             Thu May 12 09:17:39 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Desktop
 Engine:
  Version:          20.10.16
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.17.10
  Git commit:       f756502
  Built:            Thu May 12 09:15:42 2022
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.4
  GitCommit:        212e8b6fa2f44b9c21b2798135fc6fb7c53efc16
 runc:
  Version:          1.1.1
  GitCommit:        v1.1.1-0-g52de29d
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

### 步骤一 kubebuilder 初始化操作以及相应Api创建

在自己的GOPATH下面创建一个相面文件夹,相关操作查阅[kubebuilder快速开始文档](https://book.kubebuilder.io/quick-start.html)

```bash
# 项目文件夹创建
mkdir -p $GOPATH/src/kubebuilder-demo
cd $GOPATH/src/kubebuilder-demo

# 使用kubebuilder初始化,等待完成
kubebuilder init --domain demo.kubebuilder.io

# 创建API,根据提示操作
kubebuilder create api --group myapp --version v1 --kind Redis
```

查看下当前项目层级

```bash
tree -L 1
```

```text
.
├── api
├── bin
├── config
├── controllers
├── Dockerfile
├── go.mod
├── go.sum
├── hack
├── main.go
├── Makefile
├── PROJECT
└── README.md

5 directories, 7 files
```

### 步骤二 demo开发实践

主要是为了创建自定义 [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 实现自定义的 `Redis` 类型资源相关 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) 的功能:

* 定义 `CRD` 规格
* 创建 `CRD` 对应的资源 `Redis`
* 实现资源 `Redis` 创建相应 `Pod`,支持 `副本数,验证,扩容收缩`
* 添加对应事件的接收

[项目地址](https://github.com/767829413/kubebuilder-demo)

**注意事项**

1. Kubebuilder中的验证是通过 `标签` 的方式来完成

```go
// +kubebuilder:validation:Maximum:=6380
// +kubebuilder:validation:Minimum:=6370
Port int `json:"port,omitempty"`
```

[验证规则文档](https://book.kubebuilder.io/reference/markers/crd-validation.html)

2. 针对 `api/v1/xxx_types.go` 的自定义资源字段的修改,都需要执行 `make install` 来生效

```bash
## 修改后执行
make install
## 查看CRD中关于自定义字段
kubectl get crds
# NAME                              CREATED AT
# redis.myapp.demo.kubebuilder.io   2022-06-30T07:02:33Z 
kubectl get crds/redis.myapp.demo.kubebuilder.io -o yaml
```

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0.9.0
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apiextensions.k8s.io/v1","kind":"CustomResourceDefinition","metadata":{"annotations":{"controller-gen.kubebuilder.io/version":"v0.9.0"},"creationTimestamp":null,"name":"redis.myapp.demo.kubebuilder.io"},"spec":{"group":"myapp.demo.kubebuilder.io","names":{"kind":"Redis","listKind":"RedisList","plural":"redis","singular":"redis"},"scope":"Namespaced","versions":[{"name":"v1","schema":{"openAPIV3Schema":{"description":"Redis is the Schema for the redis API","properties":{"apiVersion":{"description":"APIVersion defines the versioned schema of this representation of an object. Servers should convert recognized schemas to the latest internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources","type":"string"},"kind":{"description":"Kind is a string value representing the REST resource this object represents. Servers may infer this from the endpoint the client submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds","type":"string"},"metadata":{"type":"object"},"spec":{"description":"RedisSpec defines the desired state of Redis","properties":{"num":{"type":"integer"},"port":{"maximum":6380,"minimum":6370,"type":"integer"}},"type":"object"},"status":{"description":"RedisStatus defines the observed state of Redis","type":"object"}},"type":"object"}},"served":true,"storage":true,"subresources":{"status":{}}}]}}
  creationTimestamp: "2022-06-30T07:02:33Z"
  generation: 3
  name: redis.myapp.demo.kubebuilder.io
  resourceVersion: "314435"
  uid: 2ae70d3e-7232-4c3b-ba7c-8b817dac1d09
spec:
  conversion:
    strategy: None
  group: myapp.demo.kubebuilder.io
  names:
    kind: Redis
    listKind: RedisList
    plural: redis
    singular: redis
  scope: Namespaced
  versions:
  - name: v1
    schema:
      openAPIV3Schema:
        description: Redis is the Schema for the redis API
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: RedisSpec defines the desired state of Redis
            properties:
              num:
                type: integer
              port:
                maximum: 6380
                minimum: 6370
                type: integer
            type: object
          status:
            description: RedisStatus defines the observed state of Redis
            type: object
        type: object
    served: true
    storage: true
    subresources:
      status: {}
status:
  acceptedNames:
    kind: Redis
    listKind: RedisList
    plural: redis
    singular: redis
  conditions:
  - lastTransitionTime: "2022-06-30T07:02:33Z"
    message: no conflicts found
    reason: NoConflicts
    status: "True"
    type: NamesAccepted
  - lastTransitionTime: "2022-06-30T07:02:33Z"
    message: the initial names have been accepted
    reason: InitialNamesAccepted
    status: "True"
    type: Established
  storedVersions:
  - v1
```

3. 调试方式
一个terminal执行自定义控制器

```bash
make run
```

一个terminal执行自定义资源

```bash
kubectl apply -f /项目目录/config/samples/myapp_v1_xxxx.yaml
```

两相对比进行验证
