# Kubernetes调度

## 1. 创建一个Pod工作流程

![11111111.png](https://s2.loli.net/2023/03/20/CUzLTAVhnNuJ26Y.png)

```plantuml
@startuml
actor User

User -> API_Server: create Pod
activate API_Server

API_Server-> etcd: write
activate etcd

etcd->API_Server:
deactivate

API_Server --> User:
deactivate

API_Server-> Scheduler: watch(new pod)
activate Scheduler

Scheduler->API_Server: bind pod
activate API_Server

API_Server->etcd: write
activate etcd

etcd-->API_Server:
deactivate etcd

API_Server-->Scheduler:
deactivate API_Server
deactivate Scheduler

API_Server->Kubelet: watch(bound pod)
activate Kubelet

Kubelet->Docker: docker run
activate Docker

Docker-->Kubelet:
deactivate Docker

Kubelet->API_Server: update pod status
activate API_Server

API_Server->etcd: 
activate etcd

etcd-->API_Server: 
deactivate etcd

API_Server-->Kubelet:
deactivate Kubelet
deactivate API_Server

@enduml
```

* node 上的所有组件( kubelet / kube-proxy )都是与 apiserver 通信

* master 上两个组件( scheduler / controller-manager )都是与apiserver通信

* apiserver 与其他事件产生的事件,状态都保存到 etcd

* 组件与 apiserver 周期性 watch 事件

## 2. Pod中影响调度的主要属性

## 3. 资源限制对Pod调度的影响

## 4. nodeSelector & nodeAffinity

## 5. Taints & Tolerations

## 6. nodeName 

## 7. DaemonSet控制器

## 8. 调度失败原因分析
