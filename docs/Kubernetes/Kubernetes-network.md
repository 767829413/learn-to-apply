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

    ![ClusterIP](https://pic.imgdb.cn/item/641d11caa682492fcc0f2f5f.png)

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

    ![NodePort](https://pic.imgdb.cn/item/641d11caa682492fcc0f2fc5.png)

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

    ![LoadBalancer](https://pic.imgdb.cn/item/641d11caa682492fcc0f2f7a.png)

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

NodePort的不足:

* 一个端口只能一个服务使用,端口需要提前规划
* 只支持4层负载均衡

4层: 只考虑ip和端口
7层: 协议多,过滤转发规则多

## 7.Pod与Ingress的关系

* 通过Service相关联
* 通过Ingress Controller实现Pod的负载均衡
  * 支持TCP/UDP 4层和HTTP 7层

![关系图](https://pic.imgdb.cn/item/641d0feaa682492fcc0bcafd.png)

## 8.Ingress Controller

![Ingress Controller](https://pic.imgdb.cn/item/641d1340a682492fcc118acf.png)

1. 部署Ingress Controller
2. 创建Ingress规则

Ingress Controller有很多实现，我们这里采用官方维护的Nginx控制器

Github：<https://github.com/kubernetes/ingress-nginx>

```bash
# 部署
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
```

注意事项：

* 建议直接宿主机网络暴露：hostNetwork: true

  user -> nodeport -> [iptable | ipvs] -> ingress controller(pod) node:port -> [service] -> pod

  user -> lb -> ingress controller(pod) node:port -> [service] -> pod

其他主流控制器：

Traefik： HTTP反向代理、负载均衡工具

Istio：服务治理，控制入口流量

## 9.Ingress

http:

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: example.ctnrs.com
    http:
      paths:
      - path: /
        backend:
          serviceName: web
          servicePort: 80
```

https:

1. 将证书保存到k8s secret里面
kubectl create secret-tls blog-ctnrs-com --cert=blog.ctnrs.com.pem --key=blog.ctnrs.com-key.pem

2. ingress规则里引用该secret

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  tls: 
  - hosts: 
    - sslexample.ctnrs.com
    secreName: secret-tls
  rules:
  - host: sslexample.ctnrs.com
    http:
      paths:
      - path: /
        backend:
          serviceName: web
          servicePort: 80
```

lb -> nodeport(30001,30002)
lb -> ingress controller(80,443)

解决高可用：

1. 扩容副本数
   * 提高并发能力
   * 尽量让多个节点提供服务
2. 把控制器固定到几台节点
   * daemonset（与nodeport一样）
   * nodeselector+污点
    污点，即使加污点容忍也不会完全分配到专门几个节点