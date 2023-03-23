# Kubernetes网络

## 1.Service存在的意义

`label: 关联资源,筛选资源`
`namespace: 隔离资源`

* 防止Pod失联（服务发现）
* 定义一组Pod的访问策略（负载均衡）

pod多副本特点: 分布式,高可用

场景:

前端: 展示数据
后端: 数据接口->db

1. 前端如何定位后端pod? 服务发现
2. 前端假设找到后端pod,那如何选择? 负载均衡

## 2.Pod与Service的关系

![关系](https://pic.imgdb.cn/item/641b0fd0a682492fcc0a1da4.png)

* 通过label-selector相关联
* 通过Service实现Pod的负载均衡（ TCP/UDP 4层）

## 3.Service类型

* ClusterIP：集群内部使用,通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。 这也是你没有为服务显式指定 type 时使用的默认值。 你可以使用 Ingress 或者 Gateway API 向公众暴露服务。

    ![1111111111147.png](https://s2.loli.net/2023/03/23/yvH4i3nlS6VZ1Fd.png)

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: web
  name: web
spec:
  ports:
  - port: 80 # 集群内部端口
    protocol: TCP # 协议类型
    targetPort: 80 # 容器端口
    nodePort: 31234 # 节点端口,类型为 NodePort 指定
  selector: # 标签选择器,关联对应pod
    app: web
    project: blog
  type: NodePort # 指定类型 ClusterIP | NodePort | LoadBalancer | ExternalName
```

* NodePort：对外暴露应用,通过每个节点上的 IP 和静态端口（NodePort）暴露服务。 为了让节点端口可用，Kubernetes 设置了集群 IP 地址，这等同于你请求 type: ClusterIP 的服务。

    ![43434343444409.png](https://s2.loli.net/2023/03/23/lKpP9OWcyT4aosB.png)

```yaml
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    # 默认情况下，为了方便起见，`targetPort` 被设置为与 `port` 字段相同的值。
    - port: 80
      targetPort: 80
      # 可选字段
      # 默认情况下，为了方便起见，Kubernetes 控制平面会从某个范围内分配一个端口号（默认：30000-32767）
      nodePort: 30007
```

* LoadBalancer：对外暴露应用，适用公有云,使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 NodePort 服务和 ClusterIP 服务上。

    ![eweeeeeeeee73025.png](https://s2.loli.net/2023/03/23/rUC7RcOLp9Bz4VF.png)

```yaml
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

* ExternalName(不常用)：通过返回 CNAME 记录和对应值，可以将服务映射到 externalName 字段的内容（例如，foo.bar.example.com）。 无需创建任何类型代理。

```yaml
spec:
  type: ExternalName
  externalName: my.database.example.com
```

port: service端口,集群内部访问端口
targetPort: 容器监听的端口,即应用程序端口

kube-proxy: 帮助service实现了相应的功能

用户 => LB(公网) => node:port => [service] => pod

1. 把内网节点端口提供的服务暴露到公网
2. 为NodePort提供高可用能力

## 4.Service代理模式

* IPTABLES
  * 灵活,功能强大
  * 规则遍历匹配和更新,是线性时延

    入口流量规则->轮询Pod机制->实际DNAT规则->容器
    
    ![IPTABLES](https://pic.imgdb.cn/item/641c3c7da682492fccdba86b.png)

* IPVS
  * 性能更高
  * 调度算法丰富: rr, wrr, lc, wlc, ip hash

    类似IPTABLES
    ![IPVS](https://pic.imgdb.cn/item/641c3c7da682492fccdba829.png)

## 5.Service DNS名称

DNS服务监视Kubernetes API，为每一个Service创建DNS记录用于域名解析

ClusterIP A记录格式：<service-name>.<namespace-name>.svc.cluster.local
示例：my-svc.my-namespace.svc.cluster.local

内部: pod -> coredns service -> coredns(service : clusterip记录) -> 响应A记录结果

外部: pod -> coredns service -> dns(宿主机) -> 响应

1. 采用NodePort对外暴露应用，前面加一个LB实现统一访问入口
2. 优先使用IPVS代理模式
3. 集群内应用采用DNS名称访问, 示例：my-svc.my-namespace.svc.cluster.local

## 6.Ingress为弥补NodePort不足而生

## 7.Pod与Ingress的关系

## 8.Ingress Controller

## 9.Ingress