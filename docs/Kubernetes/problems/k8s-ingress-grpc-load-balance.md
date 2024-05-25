# 使用 Ingress 解决 gRPC 长连接负载不均衡的问题

## 背景

当前服务集群的外部访问入口及负载均衡使用的是阿里云的SLB,然而其负载均衡,由于grpc长连接的缘故,表现不尽如人意,尽管加了定时重连的相关配置,多个pod依然会有旱的旱死,涝的涝死的情况.

刚好新建的k8s集群版模型服务这种情况也比较明显,所以相关的优化就被提上了日程.听说新版的nginx-ingress已经对grpc有了较好的支持,于是他来了...

本文将通过小白入门的方式，通过其工作原理，操作上手，相关配置，访问方式，以及如何解决我们实际中遇到的问题等方面详细的介绍下Ingress相关的一些特性

## Ingress简介

在Kubernetes集群中，Ingress作为集群内服务对外暴露的访问接入点，其几乎承载着集群内服务访问的所有流量。Ingress是Kubernetes中的一个资源对象，用来管理集群外部访问集群内部服务的方式。我们可以通过Ingress资源来配置不同的转发规则，从而达到根据不同的规则设置访问集群内不同的Service所对应的后端Pod。

Ingress对我们来说其实应该并不陌生,尤其是服务端的研发同学.然而我们大多数使用Ingress的场景都是鉴于其对HTTP协议的强大路由支持和负载均衡的能力,可能有不少同学也会像我一样,在此之前对其对于gRPC的支持能力和水平要打上一个圆满的问号,尤其是针对长连接的负载均衡,**真的可以做到基于请求级别的轮询吗?**

## 工作原理

为了使Nginx Ingress资源正常工作，集群中必须要有个Nginx Ingress Controller来解析Nginx Ingress的转发规则。Nginx Ingress Controller收到请求，匹配Nginx Ingress转发规则转发到后端Service所对应的Pod，由Pod处理请求。Kubernetes中Service、Nginx Ingress与Nginx Ingress Controller有着以下关系：

+ **Service是后端真实服务(可以理解为具体的pod)的抽象**，一个Service可以代表多个相同的后端服务。
+ **Nginx Ingress是反向代理规则**，用来规定HTTP/HTTPS请求应该被转发到哪个Service所对应的Pod上。例如根据请求中不同的Host和URL路径，让请求落到不同Service所对应的Pod上。
+ **Nginx Ingress Controller是一个反向代理程序**(可以理解为套了壳的nginx)，负责解析Nginx Ingress的反向代理规则。如果Nginx Ingress有增删改的变动，Nginx Ingress Controller会及时更新自己相应的转发规则，当Nginx Ingress Controller收到请求后就会根据这些规则将请求转发到对应Service的Pod上。

Nginx Ingress Controller通过API Server获取Ingress资源的变化，动态地生成Load Balancer（例如Nginx）所需的配置文件（例如nginx.conf），然后重新加载Load Balancer（例如执行`nginx -s load`重新加载Nginx）来生成新的路由转发规则。

![img](https://tech.qimao.com/content/images/2022/09/nginx-ingress.png)

Nginx Ingress Controller可通过配置LoadBalancer类型的Service来创建SLB，因此可以从外部通过SLB访问到Kubernetes集群的内部服务。根据Nginx Ingress配置的不同规则来访问不同的服务。

## Nginx Ingress一些特点

+ 支持金丝雀部署,蓝绿部署等
+ 支持网关高度定制化场景,类似原生nginx一样所有参数可配置
+ 提供七层流量处理能力与丰富的高级路由功能。
+ 强大的路由功能
  + 基于内容、源IP的路由。
  + 支持HTTP标头改写、重定向、重写、限速、跨域、会话保持等。
  + 支持请求方向转发规则和响应方向转发规则，其中响应方向转发规则可以通过扩展Snippet配置实现。
+ 支持HTTP,HTTPS,WebSocket,WSS和gRPC等协议
+ 非证书变更时可支持配置的热更新
+ 具有较好的可观测能力
  + 支持通过Access Log采集日志。
  + 支持通过Prometheus进行监控和告警配置
+ 较好的运维能力.自行维护组件,可配置HPA进行扩缩容等
+ 良好的服务治理能力
  + 服务发现支持K8s。
  + 服务灰度支持金丝雀。
  + 服务高可用支持限流。

## 操作上手

### 安装Nginx Ingress Controller组件

如果是使用阿里的ack集群,可直接在"运维管理"-"组件管理"-"核心组件"里选择该组件直接进行安装,默认且推荐安装在kube-system命名空间下

![img](https://tech.qimao.com/content/images/2022/09/1662014473468.png)

如果是自主搭建的K8s集群,可以去[GitHub](https://github.com/kubernetes/ingress-nginx)下载对应的插件版本,直接安装配置即可,这里就不展开了.

安装完成后,默认会在kube-system下生成一个名为`nginx-ingress-controller`的`deployment`,以及一个对外的`service LoadBalancer`,默认叫`nginx-ingress-lb` ,其所负责的职责就是对外提供访问入口,对内将请求负载到`nginx-ingress-controller`

### 创建Ingress

安装的部分通过我们强大运维大佬们都能很快的响应和支持,一般不需要我们操心.而我们主要关注的部分,就是`nginx-ingress`资源(大致就相当于nginx.conf),通过配置`Ingress`资源,我们可以进行相应的路由匹配,负载均衡,请求转发等操作,大体上跟`nginx`类似

#### 两种创建`Ingress`的方式

##### Web方式

第一种,可以通过web的方式创建,如阿里的ack里,可以在"网络"-"路由"里新建

![img](https://tech.qimao.com/content/images/2022/09/1662029793314.png)

##### 通过yaml方式创建

如下:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/proxy-read-timeout: '1'
    nginx.ingress.kubernetes.io/proxy-send-timeout: '1'
    nginx.ingress.kubernetes.io/service-weight: ''
  name: rec-gateway-ingress
  namespace: rec-test1
  labels:
    name: rec-gateway-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: rec-test1.demo.com
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              service:
                name: rec-gateway-headless
                port:
                  number: 9090
  tls:
    # This secret must exist beforehand
    # The cert must also contain the subj-name grpctest.dev.mydomain.com
    # https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/PREREQUISITES.md#tls-certificates
    - secretName: demo.com
      hosts:
        - rec-test1.demo.com
```

+ `name`：Ingress的名称，本例为rec-gateway-ingress。
+ `host`：指定服务访问域名。
+ `path`：指定访问的URL路径。SLB将流量转发到`backend`之前，所有的入站请求都要先匹配`host`和`path`。
+ `backend`：由服务名称和服务端口组成。
+ 服务名称：Ingress转发的`backend`服务名称。
+ 服务端口：服务暴露的端口。

上面的配置中,重点需要关注的是`nginx.ingress.kubernetes.io/backend-protocol: "GRPC"`,这个是使nginx支持grpc的基础.且由于grpc是基于`HTTP2.0`的,且`nginx Ingress controller`只运行在`HTTPS`端口上,所以`tls`证书也必须要配置.具体申请和配置证书什么的可以让运维大佬们配合,方法可在网上自行查找.

`nginx.ingress.kubernetes.io/ssl-redirect: "true"`可以使请求重定向到加密协议.

`rules`就是我们主要关注的东西了,其相当于`nginx`里的`location`

### 路由规则

路由规则是我们通过URL请求不同后端服务所必不可少的东西,通过nginx-Ingress强大的路由规则,可以满足我们针对规则转发的各种想象

```yaml
...
rules:
    - host: rec-test1.demo.com
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              service:
                name: rec-gateway-headless
                port:
                  number: 9090
```

#### pathType

Ingress 中的每个 `path` 都需要有一个对应的 `pathType`，共有三种类型：

1. `ImplementationSpecific`: 这种类型的路径匹配规则依赖具体的 Ingress Controller 实现，具体实现可以将此类型作为一个单独的类型来对待，也可以将其视为 `Prefix` 或 `Exact` 类型
2. `Exact`: 完全匹配，区分大小写
3. `Prefix`: 按前缀匹配，区分大小写，前缀部分（可补充或去掉结尾的 `/`）需完全匹配,这个也是**默认类型**

我们一般就选`Prefix`即可.

#### 正则匹配和重定向

可以通过增加 `nginx.ingress.kubernetes.io/use-regex: "true"`注解的方式开启正则路由匹配,如

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: "/$2"
  name: rec-{{ .Values.app.name }}-ingress-api
  namespace: {{ .Values.namespace }}
  labels:
    name: rec-{{ .Values.app.name }}-ingress-api
spec:
  ingressClassName: nginx
  rules:
    - host: {{ .Values.app.name }}-{{ .Values.namespace }}.wtzw.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: rec-{{ .Values.app.name }}-headless
                port:
                  number: 8080
```

`nginx-Ingress-controller`会自动将path里的正则表达式转换为location里的路由规则,同时还可以通过`nginx.ingress.kubernetes.io/rewrite-target: /$2`注解的方式对请求进行重定向.这在通过路由前缀转发请求到后端但又不想对后端服务暴露该路由前缀时会很有用

#### backend

backend就相当于我们nginx里的`proxy_pass`,`fastcgi_pass`等,将匹配到路由的请求转发到对应的后端服务.在k8s中,这些后端服务就是各个指向具体服务的service,可以是另一个`SLB`,也可以是`NodePort`,`Headless`,甚至是`hostNetwork`等,这里我比较推荐使用`headless`,成本低,效果好,可以做到基于`gRPC`长连接级别的轮询

### 通过helm管理

在k8s集群里,一般`deployment`,`service`,`ingress`等都是由我们业务方自己来维护的,而`helm`能对这些资源的管理有着更好的支持,当然这个不在我们这次的讨论范围之内,有兴趣的可以主动去了解下.我比较推荐将`deployment`,`service`,`ingress`放到一个文件进行管理,刚好yaml对这种方式也有较好的支持,如下

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: {{ .Values.namespace }}
  name: {{ .Values.app.name }}
  labels:
    app: {{ .Values.app.name }}
    chart: {{ .Chart.Name }}
    instance: {{ .Release.Name }}
    managed-by: {{ .Release.Service }}
spec:
...
{{- if .Values.service.headless }}
---
# 配置 headless service
apiVersion: v1
kind: Service
metadata:
  namespace: {{ .Values.namespace }}
  name: rec-{{ .Values.app.name }}-headless
  labels:
    app: rec-{{ .Values.app.name }}-headless
spec:
  clusterIP: None
  selector:
    app: {{ .Values.app.name }}
    instance: {{ .Release.Name }}
  type: ClusterIP
  ports:
    - name: grpc
      port: {{ .Values.app.ports.grpc }}
      protocol: TCP
      targetPort: {{ .Values.app.ports.grpc }}
{{- end }}

{{- if .Values.service.ingress }}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/proxy-read-timeout: '1'
    nginx.ingress.kubernetes.io/proxy-send-timeout: '1'
    nginx.ingress.kubernetes.io/service-weight: ''
  name: rec-{{ .Values.app.name }}-ingress
  namespace: {{ .Values.namespace }}
  labels:
    name: rec-{{ .Values.app.name }}-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: {{ .Values.app.name }}-{{ .Values.namespace }}.wtzw.com
      http:
        paths:
          - path: /
            pathType: ImplementationSpecific
            backend:
              service:
                name: rec-{{ .Values.app.name }}-headless
                port:
                  number: 9090
  tls:
    # This secret must exist beforehand
    # The cert must also contain the subj-name grpctest.dev.mydomain.com
    # https://github.com/kubernetes/ingress-nginx/blob/master/docs/examples/PREREQUISITES.md#tls-certificates
    - secretName: wtzw.com
      hosts:
        - {{ .Values.app.name }}-{{ .Values.namespace }}.wtzw.com
```

### 如何访问

#### 公网SLB访问

如果是通过阿里云容器服务管理控制台创建的Kubernetes集群在初始化时会自动部署一套Nginx Ingress Controller，默认其挂载在公网SLB实例上,请求时绑定ip访问即可

![img](https://cdn.jsdelivr.net/gh/vikiea/my_pic@master/uPic/2022/09/02/1662112707467.png)

#### 私网SLB访问

如果不同服务在同一个VPC下,也可以通过内网的方式进行直接访问,可以提供更高的效率和性能.我们可以修改(或者让运维配合)将公网的SLB调整为仅同VPC下可访问,重点增加`service.beta.kubernetes.io/alicloud-loadbalancer-address-type: intranet`注解

![img](https://cdn.jsdelivr.net/gh/vikiea/my_pic@master/uPic/2022/09/02/1662113162686.png)

#### 同时使用公网和私网SLB访问

当然,有时候我们可能期望集群内的服务既能允许公网访问，同时又希望能被同一个VPC下的其他服务直接访问（不经过公网）,这个时候就需要两者兼顾了.不过也很简单,额外部署一个`kube-system/nginx-ingress-lb-intranet`服务就行了

![img](https://cdn.jsdelivr.net/gh/vikiea/my_pic@master/uPic/2022/09/02/1662113152375.png)

### 如何建立GRPC连接

针对http请求,直接绑定slb的IP,通过host提供的域名以及路由直接请求访问即可,那么对于grpc请求,我们该如何处理呢?

#### 使用bloomRPC

如图,仅需开启TLS传输,并选择服务端证书即可

![image-20220902180926495](https://cdn.jsdelivr.net/gh/vikiea/my_pic@master/uPic/2022/09/02/1662113366641.png)

使用postman也是如此.

#### 通过SDK连接

端口选择443,传递客户端证书,如果server端关闭了校验客户端证书,则直接传空证书即可(空证书不等于空)

```go
var target = "rec-test1.demo.com:443"
var credential = insecure.NewCredentials()
if o.enabledSecurity {
  credential = credentials.NewTLS(nil)
}

dialOptions := []grpc.DialOption{
  grpc.WithTransportCredentials(credential),
  grpc.WithWriteBufferSize(writeBufSize),
  grpc.WithReadBufferSize(readBufSize),
  grpc.WithDefaultCallOptions(
    grpc.MaxCallRecvMsgSize(maxRecvBytes),
    grpc.MaxCallSendMsgSize(maxSendBytes),
  ),
  grpc.WithDefaultServiceConfig(`{"loadBalancingConfig": [{"round_robin":{}}]}`),
  grpc.WithChainUnaryInterceptor(chain...),
  grpc.WithKeepaliveParams(keepalive.ClientParameters{
    Time:                10 * time.Second,
    Timeout:             time.Second,
    PermitWithoutStream: true,
  }),
}
conn, err := grpc.DialContext(ctx, target, dialOptions...)
```

## Ingress与SLB比较

| 服务 | 优点 | 缺点 |
| --- | --- | --- |
| SLB | 1, 本质是一套转发规则,不占用实际物理资源<br>2, 仅需一个公网IP即可对外提供访问入口,成本很低 | 1, 对于长连接支持不够友好,无法较好的做到负载均衡.如新pod起来时,由于长连接未断开,会造成请求很少转发到新的pod<br>2, 对于滚动更新的响应不够及时.当服务滚动升级时,可能会由于SLB的endpoints列表没有及时更新,长连接未断开等原因导致无法平滑升级<br>3, 对外暴露IP和端口,不够直观,且安全性较低 |
| Ingress | 1, 较好的负载均衡<br>2, 路由转发功能强大,一个Ingress可以替代多个service的功能<br>3, 人类友好的方式访问<br>4, 其他优点见前面提到的内容 | 1, 需要部署一套Ingress-controller,会占用物理资源,增加一些维护成本<br>2, 在高负载的场景下需要手动调优,对维护人员要求较高 |

## **负载均衡对比**

下面验证一下其是否真的是基于请求级别的负载均衡.

前置条件:4个pod,1个client连接,分别以slb和Ingress的方式请求同一接口,持续70秒,看请求具体的响应情况

### **SLB请求**

![img](https://tech.qimao.com/content/images/2022/09/1662346031745.png)

如图,由于client设置了长连接的活跃周期是60秒,在60秒内,请求只落在gateway-5bfbd6686f-c4vfg，一分钟后重新选择，又只落在gateway-5bfbd6686f-fvn2m上,虽然在连接数较多的时候也可能达到长连接的负载均衡，但是多次实验中,发现其在重连接时很可能会与先前的pod重新建立连接,另外由于我们的服务是对内的,实际的连接数并不多,所以这种负载均衡不能给我们带来很好的保障,如下图:

![img](https://tech.qimao.com/content/images/2022/09/1662347027937.png)

### **Ingress请求**

![img](https://tech.qimao.com/content/images/2022/09/1662347107480.png)

替换成Ingress请求后,发现其始终是均匀的,所有的pod都可以做到雨露均沾.且经试验新增或者删除pod时,请求也能平滑过渡,而阿里的SLB会出现偶尔的请求不可用问题,这是由于其endpoints没有及时更新或与ipvs不同步导致的.

**至此验证了Ingress对于基于gRPC的长连接确实做到了请求级别的轮询支持**，也基本解决了我们服务请求负载不均衡的问题

## 总结

Ingress是一套经过了时间的考验非常成熟的解决方案,nginx-Ingress又依托于nginx的强大历史背景,其性能和带来的价值都是让人值得称赞的.推荐引擎此次接入Ingress,不但可以解决让我们头疼已久的负载均衡问题,而且让我们节省了多个SLB的公网IP开销,同时新的通过域名请求的方式,对我们日常开发,性能问题排查,需求测试等都带来了较大的便利

## 参考链接

[通过Ingress Controller实现gRPC服务访问](https://help.aliyun.com/document_detail/313328.html)  
[Nginx Ingress最佳实践](https://help.aliyun.com/document_detail/399745.htm)  
[Nginx路由匹配规则](https://kubernetes.github.io/ingress-nginx/user-guide/ingress-path-matching/)  
[Nginx-controller相关注解](https://help.aliyun.com/document_detail/86531.htm)  
[Ingress支持GRPC](https://github.com/kubernetes/ingress-nginx/blob/main/docs/examples/grpc/README.md)