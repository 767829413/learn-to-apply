# Ingress配置HTTPS证书安全通信

`Ingress介绍`

```text
Ingress：K8s中的一个抽象资源，给管理员
提供一个暴露应用的入口定义方法
Ingress Controller：根据Ingress生成具体
的路由规则，并对Pod负载均衡器
```

![ingress](https://pic.imgdb.cn/item/643e505d0d2dde57777e6cf4.png)

`HTTPS重要性`

```text
HTTPS是安全的HTTP，HTTP 协议中的内容都是明文传输，HTTPS 的目的是将这
些内容加密，确保信息传输安全。最后一个字母 S 指的是 SSL/TLS 协议，它位于
HTTP 协议与 TCP/IP 协议中间。
```

`HTTPS优势：`

1. 加密隐私数据：防止您访客的隐私信息(账号、地址、手机号等)被劫持或窃取。
2. 安全身份认证：验证网站的真实性，防止钓鱼网站。
3. 防止网页篡改：防止数据在传输过程中被篡改，保护用户体验。
4. 地址栏安全锁：地址栏头部的“锁”型图标，提高用户信任度。
5. 提高SEO排名：提高搜索排名顺序，为企业带来更多访问量。

`将一个项目对外暴露HTTPS访问`

配置HTTPS步骤：

1. 准备域名证书文件（来自：openssl/cfssl工具自签或者权威机构颁发）

```bash
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

cat > example.ctnrs.com-csr.json <<EOF
{
  "CN": "example.ctnrs.com",
  "hosts": [
    "example.ctnrs.com"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes example.ctnrs.com-csr.json | cfssljson -bare example.ctnrs.com
```

2. 将证书文件保存到Secret

```bash
kubectl create secret tls my-https-secret --cert=example.ctnrs.com.pem --key=example.ctnrs.com-key.pem
```

3. Ingress规则配置tls

```bash
cat > my-ing.yml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-https
spec:
  ingressClassName: "nginx"
  tls:
  - hosts:
      - example.ctnrs.com
    secretName: my-https-secret
  rules:
  - host: "example.ctnrs.com"
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: web
            port:
              number: 80
EOF

kubectl apply -f my-ing.yml
```

4. 创建暴露的deployment和service(如果没有ingress-control需要执行部署)

ingress-control部署借鉴这个:

<https://www.cnblogs.com/shanyou/p/16787191.html>

```bash
kubectl apply -f https://github.com/dotNetCloudNative/eShopOnDapr/blob/main/deploy/k8s/nginx-ingress.yaml 
```

**PS: 由于新版使用的是LoadBalancer,自己本地虚拟机搭建的可以把这个nginx-ingress.yaml中关于控制器的部分改成NodePort**

```yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.4.0
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  type: NodePort
  ipFamilyPolicy: SingleStack
  ipFamilies:
    - IPv4
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
      appProtocol: http
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
      appProtocol: https
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
```

```bash
kubectl create deployment web --image=nginx --port=80
kubectl expose deployment web --name=web --port=80 --target-port=80 --type=NodePort
kubectl get ing
NAME       CLASS   HOSTS               ADDRESS        PORTS     AGE
my-https   nginx   example.ctnrs.com   10.111.48.52   80, 443   6m11s

kubectl get svc,pod -n ingress-nginx
NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.111.48.52     <none>        80:30729/TCP,443:31100/TCP   10m
service/ingress-nginx-controller-admission   ClusterIP   10.104.179.230   <none>        443/TCP                      10m

NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-6xwcv        0/1     Completed   0          10m
pod/ingress-nginx-admission-patch-6b8sq         0/1     Completed   0          10m
pod/ingress-nginx-controller-7455cb8948-nlb65   1/1     Running     0          10m

k get ingressclasses.networking.k8s.io 
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       14m
```

80:32397/TCP => http

443:31156/TCP => https

浏览器 -> Service(Nodeport) -> ingress controller -> pod(需要暴露的)

5. 测试，本地电脑绑定hosts记录对应ingress里面配置的域名，IP是Ingress Concontroller Pod节点IP

本机的hosts文件记得添加 

192.168.148.73 example.ctnrs.com

使用 <https://example.ctnrs.com:31100/> 浏览器中请求下

![res1](https://pic.imgdb.cn/item/6440f7220d2dde57777ff75a.png)
![res2](https://pic.imgdb.cn/item/6440f7220d2dde57777ff793.png)