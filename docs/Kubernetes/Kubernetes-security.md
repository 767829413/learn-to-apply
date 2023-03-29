# Kubernetes安全

## 1. Kubernetes 安全框架

* 访问K8S集群的资源需要过三关：认证、鉴权、准入控制
* 普通用户若要安全访问集群API Server，往往需要证书、Token或者用户名+密码；Pod访问，需要ServiceAccount
* K8S安全控制框架主要由下面3个阶段进行控制，每一个阶段
都支持插件方式，通过API Server配置来启用插件。
    1. Authentication（鉴权）
    2. Authorization（授权）
    3. Admission Control（准入控制）

![框架](https://pic.imgdb.cn/item/6423d6a0a682492fcc31aee5.png)

## 2. 认证，授权，准入控制

### 鉴权(Authentication)

    三种客户端身份认证:

    * HTTPS 证书认证: 基于CA证书签名的数字证书认证
    * HTTP Token 认证: 通过一个Token来识别用户
    * HTTP Base 认证: 用户名+密码的方式认证(不推荐)

    apiserver http接口: https通信

    * 安全通信
    * 基于证书来认证(从证书拿用户名,用户组)

### 授权(Authorization)

    RBAC（Role-Based Access Control，基于角色的访问控制）：负责完成授权（Authorization）工作。
    根据API请求属性，决定允许还是拒绝。

    * user：用户名
    * group：用户分组
    * extra：用户额外信息
    * API
    * 请求路径：例如/api，/healthz
    * API请求方法：get，list，create，update，patch，watch，delete
    * HTTP请求方法：get，post，put，delete
    * 资源
    * 子资源
    * 命名空间
    * API组

### 准入控制

    Adminssion Control实际上是一个准入控制器插件列表，发送到API Server的请求都需要经过这个列表中的每个准入控制器插件的检查，检查不通过，则拒绝请求。

    1. 用户定制化插件
    2. 官方实现的一些高级插件功能

## 3. 基于角色的权限访问控制：RBAC

RBAC（Role-Based Access Control，基于角色的访问控制），允许通过Kubernetes API动态配置策略。

角色

* Role：授权特定命名空间的访问权限
* ClusterRole：授权所有命名空间的访问权限

角色绑定

* RoleBinding：将角色绑定到主体（即subject）
* ClusterRoleBinding：将集群角色绑定到主体

主体（subject）

* User：用户
* Group：用户组
* ServiceAccount：服务账号

![总体架构](https://pic.imgdb.cn/item/6423f0c7a682492fcc59efc5.png)

## 4. 案例：为指定用户授权访问不同命名空间权限

为用户aaa授权default命名空间pod读取权限

1. 用K8S CA签发客户端证书

    ```sh
    cat > ca-config.json <<EOF
    {
      "signing": {
        "default": {
          "expiry": "87600h"
        },
        "profiles": {
          "kubernetes": {
            "usages": [
                "signing",
                "key encipherment",
                "server auth",
                "client auth"
            ],
            "expiry": "87600h"
          }
        }
      }
    }
    EOF

    cat > aaa-csr.json <<EOF
    {
      "CN": "aaa",
      "hosts": [],
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "CN",
          "ST": "BeiJing",
          "L": "BeiJing",
          "O": "k8s",
          "OU": "System"
        }
      ]
    }
    EOF

    cfssl gencert -ca=/etc/kubernetes/pki/ca.crt -ca-key=/etc/kubernetes/pki/ca.key -config=ca-config.json -profile=kubernetes aaa-csr.json | cfssljson -bare aaa
    ```

    CN 用户名
    O 用户组

2. 生成kubeconfig授权文件

    ```bash
    kubectl config set-cluster kubernetes \
    --certificate-authority=/etc/kubernetes/pki/ca.crt \
    --embed-certs=true \
    --server=https://192.168.148.61:6443 \
    --kubeconfig=aaa.kubeconfig
    # 设置客户端认证
    kubectl config set-credentials aaa \
    --client-key=aaa-key.pem \
    --client-certificate=aaa.pem \
    --embed-certs=true \
    --kubeconfig=aaa.kubeconfig
    # 设置默认上下文
    kubectl config set-context kubernetes \
    --cluster=kubernetes \
    --user=aaa \
    --kubeconfig=aaa.kubeconfig
    # 设置当前使用配置
    kubectl config use-context kubernetes --kubeconfig=aaa.kubeconfig
    ```

3. 创建RBAC权限策略

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      namespace: default
      name: pod-reader
    rules:
    - apiGroups: [""]
      resources: ["pods", "services"]
      verbs: ["get", "list", "watch"]
    
    ---
    
    apiVersion: rbac.authorization.k8s.io/v1
    # 此角色绑定允许 "jane" 读取 "default" 名字空间中的 Pod
    # 你需要在该命名空间中有一个名为 “pod-reader” 的 Role
    kind: RoleBinding
    metadata:
      name: read-pods
      namespace: default
    subjects:
    # 你可以指定不止一个“subject（主体）”
    - kind: User
      name: aaa # "name" 是区分大小写的
      apiGroup: rbac.authorization.k8s.io
    roleRef:
      # "roleRef" 指定与某 Role 或 ClusterRole 的绑定关系
      kind: Role        # 此字段必须是 Role 或 ClusterRole
      name: pod-reader  # 此字段必须与你要绑定的 Role 或 ClusterRole 的名称匹配
      apiGroup: rbac.authorization.k8s.io
    ```

    ```bash
    [root@master ssl]# k --kubeconfig=aaa.kubeconfig get po
    NAME                                      READY   STATUS    RESTARTS   AGE
    mypod                                     1/1     Running   0          6h57m
    [root@master ssl]# k --kubeconfig=aaa.kubeconfig get svc
    NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   11d
    [root@master ssl]# k --kubeconfig=aaa.kubeconfig get po -n kube-system
    Error from server (Forbidden): pods is forbidden: User "aaa" cannot list resource "pods" in API group "" in the namespace "kube-system"
    ```

<https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/>
![示意图](https://pic.imgdb.cn/item/6423f612a682492fcc60c191.png)

## 5. 网络策略概述

## 6. 案例：对项目Pod出入流量访问控制