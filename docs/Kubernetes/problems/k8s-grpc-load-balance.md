# 在 Kubernetes 上对 gRPC 进行负载均衡

## 问题引申

因为最近线上因为阿里云服务器的重启,导致部分k8s集群的服务异常重启

其中使用了 ETCD 作为服务端服务注册发现中间件,本身自建的 ETCD 在重启后还是导致服务间发现失败,后面通过去掉对 ETCD 的依赖,使用基础架构层的 Kubernetes 本身的 service 做服务的注册和发现, 服务端内部访问使用的是 gRPC,这里针对这种场景做了调研和方案处理

## Kubernetes 上的 gRPC 负载平衡

Kubernetes 的 deployment 控制器是可以控制部署多个无状态的POD, 为许多客户端请求提供服务. 同时 Kubernetes 的 service 的 ClusterIP 类型服务可以提供负载均衡的 IP 地址, 使用 HTTP 作为服务间调用, 直接使用 http://serviceName:port 就可以请求, service 自身会帮我们做好负载均衡, 但是不适用于 gRPC. 

主要原因如下:

gRPC 在 HTTP/2 上工作, HTTP/2 上的 TCP 连接是长期存在的, 一个连接可以供多个请求多路复用. 所以 grpc 是长连接. 

对于在 Kubernetes 中使用 service 就相当于基于连接的负载均衡, 因为这种机制通过在网络层面上分配传入请求的连接，将流量均匀地分发到多个服务器上，以确保各服务器的负载较为均衡, 但是 HTTP/2 长连接这种机制导致 Kubernetes 的默认负载平衡不能与 gRPC 一起工作, 会导致 gRPC 请求粘连到固定的 pod 上.

为了解决这种问题采用: grpc resolver dns 模式 + 无头服务 + gRPC服务端配置 [MaxConnectionAge](https://pkg.go.dev/google.golang.org/grpc/keepalive#ServerParameters) 

### grpc resolver dns

grpc 内置三种 resolver: `passthrough`, `manual` 和 `dns`, 具体可以查看文档

> [Custom Name Resolution](https://grpc.io/docs/guides/custom-name-resolution/)

这里重点说明 `dns`

[dns_resolver.go](https://github.com/grpc/grpc-go/blob/21976fa3e38a266811384409bc8b25437cc1ff1d/resolver/dns/dns_resolver.go) `dns` 模式会在 `resolve` 阶段通过 `dns lookup` 将 `host` 解析成 `ip`, 作为 `addrs` 传入底层连接.

连接地址传入 `dns:///serviceName:port` 时, `serviceName` 会通过 `dns` 解析, 传入底层连接地址会变成 `['x.x.x.x:port']`. 但是直接使用 `serviceName` 时, 解析出的 `ip` 是 `service` 的 `ip`, 然后底层 `Dial` 时, 也通过 `service` 和一个 `pod` 建立连接, 因为解析出的 ip 不会有多个, 这样暂时做不到服务发现和负载均衡.

### 无头服务

> [无头服务-Headless Services](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#headless-services)

Kubernetes 允许客户端通过 DNS 查找来发现 pod IP。在对服务执行 DNS 查找时，DNS 服务器会返回一个 IP(服务的集群 IP). 但是如果服务不需要集群 IP（可以通过 service.spec.clusterIP: None 来实现）. 

此时 DNS 服务器将返回 pod IP 而不是单个服务 IP。因为 DNS 服务器将返回服务的多个 A 记录，而不是返回单个 DNS A 记录，每个记录都指向当时支持该服务的单个 pod 的 IP。因此，客户端可以进行简单的 DNS A 记录查找并获取属于服务的所有pod的 IP。然后，客户端可以使用该信息连接到其中一个、多个或全部。

```bash
# 普通 service
nslookup school-internal.rongke-dev-s1
服务器:  UnKnown
Address:  10.0.255.1

非权威应答:
名称:    school-internal.rongke-dev-s1
Addresses:  10.21.9.38
          10.21.9.38

nslookup school-internal.rongke-dev-s1.svc.cluster.local
服务器:  UnKnown
Address:  10.0.255.1

非权威应答:
名称:    school-internal.rongke-dev-s1.svc.cluster.local
Addresses:  10.21.9.38
          10.21.9.38

# headless service
nslookup school-internal-headless.rongke-dev-s1
服务器:  UnKnown
Address:  10.0.255.1

非权威应答:
名称:    school-internal-headless.rongke-dev-s1
Addresses:  10.20.173.182
          10.20.173.23

nslookup school-internal-headless.rongke-dev-s1.svc.cluster.local
服务器:  UnKnown
Address:  10.0.255.1

非权威应答:
名称:    school-internal-headless.rongke-dev-s1.svc.cluster.local
Addresses:  10.20.173.182
          10.20.173.23
```

可以看到解析出来的结果确实是: 普通 service 解析出 service ip, headless 解析出所有 pod 的 ip.

### gRPC服务端配置 [MaxConnectionAge](https://pkg.go.dev/google.golang.org/grpc/keepalive#ServerParameters) 

在 client 一直连接的情况下 kill 一个 pod 触发重启, ip 发生变化时, 会发现新出现的 pod 不会收到任何请求, 断开 client 重新连接时, 又会正常.

出现这种状况的原因是 grpc dns 解析会缓存解析结果, resolve 阶段之后每 30 分钟才会刷新一次, pod 下线时, grpc 会剔除掉不健康的地址, 但是新地址必须要在刷新之后或者重新连接时才能解析到. 细节查看 [grpc/grpc/issues/12295](https://github.com/grpc/grpc/issues/12295), 并且官方不认为这是个问题.

微软的 <https://dapr.io> 项目也是在 k8s 服务发现和证书过期的问题上遇到过上述问题, 他们通过设置 `gRPC server` 端 `MaxConnectionAge` 来定时踢掉 `client` 连接. 这样就不会导致因为缓存而出现 POD 重启后无法解析 ip 的问题.

### 实际演示

* Server pod
  * Kubernetes 集群中的 deployment 下的 school-internal Pod
    * school-internal-76cc44cfb5-hmt7l
    * school-internal-76cc44cfb5-j57xm

* Client
  * 本地启动一个 manage_service 进行请求

* Service
  * Kubernetes 集群中的 rongke-dev-s1/service 
    * school-internal

![img](https://pic.imgdb.cn/item/65eac4189f345e8d03b9599b.jpg)

![img](https://pic.imgdb.cn/item/65eada359f345e8d03efaa04.png)
![img](https://pic.imgdb.cn/item/65eada359f345e8d03efab25.png)

![img](https://pic.imgdb.cn/item/65eada359f345e8d03efaa96.png)
![img](https://pic.imgdb.cn/item/65eada369f345e8d03efabdd.png)

![img](https://pic.imgdb.cn/item/65eada719f345e8d03f032b4.png)
![img](https://pic.imgdb.cn/item/65eada719f345e8d03f033cc.png)

![img](https://pic.imgdb.cn/item/65eada719f345e8d03f03333.png)
![img](https://pic.imgdb.cn/item/65eada719f345e8d03f03456.png)

可以看到服务端收到请求分布到不同的pod, 也就是达到了服务发现和负载均衡效果

## 参考资料

* [gRPC load balancing on Kubernetes (using Headless Service)](https://techdozo.dev/grpc-load-balancing-on-kubernetes-using-headless-service/)