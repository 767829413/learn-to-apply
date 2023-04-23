# Kubernetes调度

## 1. 创建一个Pod工作流程

![调度流程](https://s2.loli.net/2023/03/20/CUzLTAVhnNuJ26Y.png)

* node 上的所有组件( kubelet / kube-proxy )都是与 apiserver 通信

* master 上两个组件( scheduler / controller-manager )都是与apiserver通信

* apiserver 与其他事件产生的事件,状态都保存到 etcd

* 组件与 apiserver 周期性 watch 事件

## 2. Pod中影响调度的主要属性

```yaml
spec:
  containers:
    - image: nginx
      imagePullPolicy: Always
      name: nginx
      resources: {} # 资源限制影响节点分配
  nodeName: node1 # 已经完成调度
  nodeSelector: {} # 节点选择
  affinity: {} # 节点亲和
  schedulerName: default-scheduler # 默认调度器
  tolerations: #对污点的容忍
    - effect: NoExecute
      key: node.kubernetes.io/not-ready
      operator: Exists
      tolerationSeconds: 300
    - effect: NoExecute
      key: node.kubernetes.io/unreachable
      operator: Exists
      tolerationSeconds: 300
```

## 3. 资源限制对Pod调度的影响

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

如果不配置 resources ,那 pod 可以使用宿主机所有资源,调度时不参考配额

cpu配置写法: m单位, 1 = 1000m = 1c , 0.5 = 500m = 0.5c

resources 资源限制

* requests: 资源请求值,调度pod依据,
* limits: 资源最大限制
* requests的值必须小于limits

## 4. nodeSelector & nodeAffinity

```yaml
spec:
  nodeSelector:
    disktype: hd
```

`nodeSelector`: 用于将Pod调度到匹配Label的node上

```bash
# 给节点打标签
kubectl label nodes [node] key1=value1 key2=value2
# 给节点移除标签
kubectl label nodes [node] key1- key2-
```

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: label-2
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1 # 1 ~ 100,值越大,pod调度到该标签节点可能性最大
        preference:
          matchExpressions:
          - key: label-1
            operator: In
            values:
            - key-1
```

`nodeAffinity`: 节点亲和类似于 `nodeSelector` ,可以根据节点上的标签来约束 Pod 可以调度到哪些节点

相比 `nodeSelector` :

* 匹配有更多的逻辑组合,不只是字符串的完全相等
* 调度分为软策略和硬策略,而不是硬性要求
  * 硬(required): 必须满足,字段 requiredDuringSchedulingIgnoredDuringExecution
  * 软(preferred): 尝试满足,但不保证, 字段 preferredDuringSchedulingIgnoredDuringExecution
* 操作符号: In, NotIn, Exists, DoesNotExist, Gt, Lt

## 5. Taints & Tolerations

nodeSelector & nodeAffinity: 将pod分配到某些节点,pod属性

Taints：避免Pod调度到特定Node上,节点属性

应用场景：

* 专用节点
* 配备了特殊硬件的节点
* 基于Taint的驱逐

设置污点：

```bash
kubectl taint node [node] key=value:[effect] 
```

其中[effect] 可取值：

* NoSchedule：一定不能被调度。
* PreferNoSchedule：尽量不要调度。
* NoExecute：不仅不会调度，还会驱逐Node上已有的Pod。

去掉污点：

```bash
kubectl taint node [node] key:[effect]-
```

Tolerations：允许Pod调度到有特定Taints的Node上

容忍污点: 不是强制性分配到具有污点的节点上,只是在调度时候忽略节点污点的影响

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-taints
spec:
  containers:
  - name: pod-taints
    image: busybox:latest
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

## 6. nodeName 

nodeName：用于将Pod调度到指定的Node上，不经过调度器

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-example
  labels:
    app: nginx
spec:
  nodeName: k8s-node2
  containers:
  - name: nginx
    image: nginx:1.15
```

## 7. DaemonSet控制器

![结构示意](https://pic.imgdb.cn/item/641a70b3a682492fccd6441e.png)

DaemonSet功能：

* 在每一个Node上运行一个Pod
* 新加入的Node也同样会自动运行一个Pod

应用场景：网络插件、Agent

* 监控: zabbix agent
* 日志: filebeat
* c/s架构

## 8. 调度失败原因分析

```bash
# 查看调度结果：
kubectl get pod <NAME> -o wide
# 查看调度失败原因：
kubectl describe pod <NAME>
```
