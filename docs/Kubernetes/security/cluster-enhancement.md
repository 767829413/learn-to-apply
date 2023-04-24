# 集群强化

## k8s安全框架

`介绍`

```text
K8S安全控制框架主要由下面3个阶段进行控制，每一个阶段都支持插件方式，通过API Server配置来启用插件

1. Authentication（鉴权）
2. Authorization（授权）
3. Admission Control（准入控制）

客户端要想访问k8s集群API Server,一般需要证书,token或用户名密码;
如果是Pod内访问,则需要 ServiceAccount
```

![k8s安全框架.png](https://s2.loli.net/2023/04/23/xtLpv6eMWCXQr2d.png)

`鉴权（Authentication）`

K8s Apiserver提供三种客户端身份认证：

- HTTPS 证书认证：基于CA证书签名的数字证书认证（kubeconfig）
- HTTP Token认证：通过一个Token来识别用户（serviceaccount）
- HTTP Base认证：用户名+密码的方式认证（1.19版本弃用）

`授权（Authorization）`

RBAC（Role-Based Access Control，基于角色的访问控制）：负责完成授权（Authorization）工作。

RBAC根据API请求属性，决定允许还是拒绝。

比较常见的授权维度：

- user：用户名
- group：用户分组
- 资源，例如pod、deployment
- 资源操作方法：get，list，create，update，patch，watch，delete
- 命名空间
- API组

`准入控制（Admission Control）`

```bash
# 启用一个准入控制器：
kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...
# 关闭一个准入控制器：
kube-apiserver --disable-admission-plugins=PodNodeSelector,AlwaysDeny ...
# 查看默认启用：
kubectl exec kube-apiserver-k8s-master -n kube-system -- kube-apiserver -h | grep enable-admission-plugins
```

## RBAC认证授权案例

`RBAC介绍`

```text
RBAC（Role-Based Access Control，基于角色的访问控制），是K8s默认授权策略，并且是动态配置策略（修改即时生效）
```

`主体（subject）`

- User：用户
- Group：用户组
- ServiceAccount：服务账号

`角色`

- Role：授权特定命名空间的访问权限
- ClusterRole：授权所有命名空间的访问权限角色绑定
- RoleBinding：将角色绑定到主体（即subject）
- ClusterRoleBinding：将集群角色绑定到主体

**PS: RoleBinding在指定命名空间中执行授权，ClusterRoleBinding在集群范围执行授权。**

![rbac.png](https://s2.loli.net/2023/04/23/bACU6Frkuvaimlx.png)

`k8s内置角色`

```text
k8s预定好了四个集群角色供用户使用，使用kubectl get clusterrole查看，其中systemd:开头的为系统内部使用。
```

| 内置集群角色 | 描述 |
| --- | --- |
| cluster-admin | 超级管理员，对集群所有权限 |
| admin | 主要用于授权命名空间所有读写权限 |
| edit | 允许对命名空间大多数对象读写操作，不允许查看或者修改角色、角色绑定。 |
| view | 允许对命名空间大多数对象只读权限，不允许查看角色、角色绑定和Secret |

`用户权限和用户组权限冲突`

```text
在Kubernetes中，RBAC（基于角色的访问控制）用于控制用户或者组对于Kubernetes资源的访问权限。如果发生了RBAC组和用户权限的冲突，Kubernetes将使用RBAC组的权限。

这是因为在Kubernetes中，RBAC是最终的授权决策机制。这意味着如果用户属于多个组，并且这些组中有一个组有权访问资源，那么用户将被授予访问该资源的权限。

例如，假设用户属于 GroupA 和 GroupB 两个组，而 GroupA 具有访问 Pod 的权限，而 GroupB 没有。在这种情况下，用户将被授予访问 Pod 的权限，因为 GroupA 具有访问 Pod 的权限。

因此，在Kubernetes中，RBAC组的权限优先于用户的权限。但是，为了避免冲突，最好在RBAC组和用户权限之间进行管理，以确保正确的授权和访问控制。
```

`ServiceAccount介绍`

```text
先了解下ServiceAccount，简称SA，是一种用于让程序访问K8s API的服务账号:

• 当创建namespace时，会自动创建一个名为default的SA，这个SA没有绑定任何权限
• 当default SA创建时，会自动创建一个default-token-xxx的secret，并自动关联到SA
• 当创建Pod时，如果没有指定SA，会自动为pod以volume方式挂载这个default SA，在容器目录：/var/run/secrets/kubernetes.io/serviceaccount

验证默认SA权限：kubectl --as=system:serviceaccount:default:default get pods
```

`案例1：对用户授权访问K8s（TLS证书）`

```text
需求：为指定用户授权访问不同命名空间权限，例如新入职一个小弟，希望让他先熟悉K8s集群，为了安全性，先不能给他太大权限，因此先给他授权访问default命名空间Pod读取权限

实施大致步骤：
1. 用 k8s CA 签发客户端证书
2. 生成kubeconfig授权文件
3. 创建RBAC权限策略
4. 指定kubeconfig文件测试权限：kubectl get pods --kubeconfig=./user1.kubeconfig
```

用 k8s CA 签发客户端证书

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

cat > user1-csr.json <<EOF
{
    "CN": "user1",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "system"
        }
    ]
}
EOF

cfssl gencert -ca=/etc/kubernetes/pki/ca.crt -ca-key=/etc/kubernetes/pki/ca.key -config=./ca-config.json -profile=kubernetes ./user1-csr.json | cfssljson -bare user1
```

生成kubeconfig授权文件

```bash
# 在 kubeconfig 中设置集群条目
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/pki/ca.crt \
--embed-certs=true --server=https://192.168.148.73:16443 \
--kubeconfig=user1.kubeconfig

# 设置客户端用户认证
kubectl config set-credentials user1 \
--client-key=user1-key.pem \
--client-certificate=user1.pem \
--embed-certs=true \
--kubeconfig=user1.kubeconfig

# 设置默认上下文
kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=user1 \
--kubeconfig=user1.kubeconfig

# 设置当前使用配置
kubectl config use-context kubernetes --kubeconfig=user1.kubeconfig
# 测试一下新的用户权限
kubectl get pods --kubeconfig=./user1.kubeconfig
Error from server (Forbidden): pods is forbidden: User "user1" cannot list resource "pods" in API group "" in the namespace "default"
```

创建RBAC权限策略

角色权限主体结构：

```bash
cat > rbac.yml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" 标明 core API 组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
# 此角色绑定允许 "jane" 读取 "default" 名字空间中的 Pod
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# 你可以指定不止一个“subject（主体）”
- kind: User
  name: user1 # "name" 是区分大小写的
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" 指定与某 Role 或 ClusterRole 的绑定关系
  kind: Role        # 此字段必须是 Role 或 ClusterRole
  name: pod-reader  # 此字段必须与你要绑定的 Role 或 ClusterRole 的名称匹配
  apiGroup: rbac.authorization.k8s.io
EOF
```

将主体与角色绑定：

```bash
kubectl apply -f rbac.yml
```

指定kubeconfig文件测试权限：

```bash
# 可以访问到pod资源
kubectl get pods --kubeconfig=./user1.kubeconfig
No resources found in default namespace.
# 访问其他资源禁止
kubectl get svc --kubeconfig=./user1.kubeconfig
Error from server (Forbidden): services is forbidden: User "user1" cannot list resource "services" in API group "" in the namespace "default"
```

**用户组：用户组的好处是无需单独为某个用户创建权限，统一为这个组名进行授权，所有的用户都以组的身份访问资源,组和用户权限冲突时候以用户权限为主**

**例如：为dev用户组统一授权**

```text
1. 将certs.sh文件中的user1-csr.json下的O字段改为dev，并重新生成证
书和kubeconfig文件
2. 将dev用户组绑定Role（pod-reader）
3. 测试，只要O字段都是dev，这些用户持有的kubeconfig文件都拥有相
同的权限
```

修改下RoleBinding内容:

```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: Group
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

![tls-rbac.png](https://s2.loli.net/2023/04/23/Sh7FxsPWlCqOmBH.png)

`案例2：对应用程序授权访问K8s（ServiceAccount）`

```text
需求：授权容器中Python程序对K8s API访问权限
实施大致步骤：

1. 创建Role
2. 创建ServiceAccount
3. 将ServiceAccount与Role绑定
4. 为Pod指定自定义的SA
5. 进入容器里执行Python程序测试操作K8s API权限
```

创建Role

```bash
# 创建角色
kubectl create role py-role --verb=get,watch,list \
--resource=pods -n test
# 对应yml文件
cat > sa-role.yml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test
  name: py-role
rules:
- apiGroups: [""] # "" 标明 core API 组
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
EOF
```

创建ServiceAccount

```bash
kubectl create serviceaccount py-k8s -n test
# 对应yml文件
cat sa.yml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: py-k8s
  namespace: test
EOF
```

将ServiceAccount与Role绑定

```bash
kubectl create rolebinding py-bind \
--serviceaccount=test:py-k8s --role=py-role -n test
# 对应yml文件
cat > sa-rolebinding.yml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: py-bind
  namespace: test
subjects:
- kind: ServiceAccount
  name: py-k8s
roleRef:
  kind: Role
  name: py-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

测试下创建的sa权限

```bash
kubectl --as=system:serviceaccount:test:py-k8s get pods -n test
```

为Pod指定自定义的SA

```bash
cat > sa-pod.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: py-k8s
  namespace: test
spec:
  serviceAccountName: py-k8s 
  containers:
  - image: python:3
    name: python
    command:
    - sleep 
    - 24h
EOF

kubectl apply -f sa-pod.yml

cat > test.py <<EOF
from kubernetes import client, config

config.load_incluster_config()
apps_api = client.AppsV1Api() 
core_api = client.CoreV1Api() 
try:
  print("###### Deployment列表 ######")
  #列出default命名空间所有deployment名称
  for dp in apps_api.list_namespaced_deployment("test").items:
    print(dp.metadata.name)
except:
  print("没有权限访问Deployment资源！")

try:
  #列出default命名空间所有pod名称
  print("###### Pod列表 ######")
  for po in core_api.list_namespaced_pod("test").items:
    print(po.metadata.name)
except:
  print("没有权限访问Pod资源！")
EOF

kubectl cp test.py test/py-k8s:/root
kubectl exec -it py-k8s -n test -- python /root/test.py
###### Deployment列表 ######
没有权限访问Deployment资源！
###### Pod列表 ######
nginx-ff6774dc6-8g9db
py-k8s
```

![sa.png](https://s2.loli.net/2023/04/24/NLMVze6T2Ai97mG.png)

## 资源配额 ResourceQuota

`介绍`

```text
当多个团队、多个用户共享使用K8s集群时，会出现不均匀资源使用，默认情况下先到先得，这时可以通过
ResourceQuota来对命名空间资源使用总量做限制，从而解决这个问题。

使用流程：k8s管理员为每个命名空间创建一个或多个ResourceQuota对象，定义资源使用总量，K8s会跟踪命名空间资源使用情况，当超过定义的资源配额会返回拒绝。
```

ResourceQuota功能是一个准入控制插件，默认已经启用

| 支持的资源 | 描述 |
| --- | --- |
| limits.cpu/memory | 所有Pod上限资源配置总量不超过该值（所有非终止状态的Pod） |
| requests.cpu/memory | 所有Pod请求资源配置总量不超过该值（所有非终止状态的Pod） |
| cpu/memory | 等同于requests.cpu/requests.memory |
| requests.storage | 所有PVC请求容量总和不超过该值 |
| persistentvolumeclaims  | 所有PVC数量总和不超过该值 |
| <storage-class-name>.storageclass.storage.k8s.io/requests.storage | 所有与<storage-class-name>相关的PVC请求容量总和不超过该值 |
| <storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims | 所有与<storage-class-name>相关的PVC数量总和不超过该值 |
| pods、count/deployments.apps、count/statfulsets.apps、count/services（services.loadbalancers、services.nodeports）、count/secrets、count/configmaps、count/job.batch、count/cronjobs.batch | 创建资源数量不超过该值 |

`资源配额的格式和使用`

查看配额操作

```bash
kubectl get quota -n test
```

```bash
# 计算资源配额
cat <<EOF > compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: test
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "1.5"
    limits.memory: 2Gi
EOF
kubectl apply -f compute-resources.yaml
# 测试一下
cat > test-pod1.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod1
  namespace: test
spec:
  containers:
  - name: test-pod1
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "1.6"
      requests:
        memory: "200Mi"
        cpu: "0.7"
EOF
kubectl apply -f test-pod1.yml
Error from server (Forbidden): error when creating "test-pod1.yml": pods "test-pod1" is forbidden: exceeded quota: compute-resources, requested: limits.cpu=1600m, used: limits.cpu=0, limited: limits.cpu=1500m

# 存储资源配额
cat > storage-resources.yml <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-resources
  namespace: test
spec:
  hard:
    requests.storage: "10G"
    nfs-client.storageclass.storage.k8s.io/requests.storage: "5G"
EOF
kubectl apply -f storage-resources.yml
# 测试一下
cat > test-pv1.yml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pv1
  namespace: test
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 13Gi
EOF
kubectl apply -f test-pv1.yml
Error from server (Forbidden): error when creating "test-pv1.yml": persistentvolumeclaims "test-pv1" is forbidden: exceeded quota: compute-resources, requested: nfs-client.storageclass.storage.k8s.io/requests.storage=13Gi,requests.storage=13Gi, used: nfs-client.storageclass.storage.k8s.io/requests.storage=0,requests.storage=0, limited: nfs-client.storageclass.storage.k8s.io/requests.storage=5G,requests.storage=10G

# 对象数量配额
cat > object-resources.yml <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-resources
  namespace: test
spec:
  hard:
    pods: "3"
    count/deployments.apps: "2"
    count/services: "1"
EOF
kubectl apply -f object-resources.yml
kubectl create deployment test --replicas 4 --image nginx -n test
kubectl get rs -n test
NAME              DESIRED   CURRENT   READY   AGE
test-865bcfc74c   4         3         3       30s
k describe rs test-865bcfc74c -n test
...
Events:
  Type     Reason            Age                 From                   Message
  ----     ------            ----                ----                   -------
  Normal   SuccessfulCreate  43s                 replicaset-controller  Created pod: test-865bcfc74c-t79mq
  Normal   SuccessfulCreate  43s                 replicaset-controller  Created pod: test-865bcfc74c-t45hd
  Normal   SuccessfulCreate  43s                 replicaset-controller  Created pod: test-865bcfc74c-6m8lw
  Warning  FailedCreate      43s                 replicaset-controller  Error creating: pods "test-865bcfc74c-7x8zx" is forbidden: exceeded quota: object-resources, requested: pods=1, used: pods=3, limited: pods=3
...
```

## 资源限制 LimitRange

`介绍`

```text
默认情况下，K8s集群上的容器对计算资源没有任何限制，可能会导致个别容器资源过大导致影响其他容器正常工作，这时可以使用LimitRange定义容器默认CPU和内存请求值或者最大上限。

LimitRange限制维度：

- 限制容器配置requests.cpu/memory，limits.cpu/memory的最小、最大值
- 限制容器配置requests.cpu/memory，limits.cpu/memory的默认值
- 限制PVC配置requests.storage的最小、最大值
```

`资源限制的格式和使用`

查看资源限制操作

```bash
kubectl get limits -n test
kubectl describe limits -n test
```

```bash
# 计算资源最大、最小限制
cat > cpu-memory-max-demo-lr.yml <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-memory-max-demo-lr
  namespace: test
spec:
  limits:
  - max: # 容器能设置limit的最大值
      cpu: "800m" 
      memory: 1Gi
    min: # 容器能设置request的最小值
      cpu: "200m"
      memory: 500Mi
    type: Container
EOF
kubectl apply -f cpu-memory-max-demo-lr.yml
# 测试一下
cat > test-pod2.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod2
  namespace: test
spec:
  containers:
  - name: test-pod2
    image: nginx
    resources:
      limits:
        memory: "900Mi"
        cpu: "0.8"
      requests:
        memory: "100Mi"
        cpu: "0.2"
EOF
kubectl apply -f test-pod2.yml
Error from server (Forbidden): error when creating "test-pod2.yml": pods "test-pod2" is forbidden: minimum memory usage per Container is 500Mi, but request is 100Mi

# 计算资源默认值限制
cat > cpu-memory-default-demo-lr.yml <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-memory-default-demo-lr
  namespace: test
spec:
  limits:
  - default:
      cpu: 0.5
      memory: "200Mi"
    defaultRequest:
      cpu: 0.1
      memory: "100Mi"
    type: Container
EOF
kubectl apply -f cpu-memory-default-demo-lr.yml
# 测试一下
k run test --image nginx -n test
kubectl describe limits -n test
...
    Limits:
      cpu:     500m
      memory:  200Mi
    Requests:
      cpu:        100m
      memory:     100Mi
...
# 存储资源最大、最小限制
cat > storage-demo-lr.yml <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: storage-demo-lr
  namespace: test
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 2Gi
    min:
      storage: 1Gi
EOF
kubectl apply -f storage-demo-lr.yml
# 测试一下
cat > test-pvc2.yml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc2
  namespace: test
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
EOF
kubectl apply -f test-pvc2.yml
Error from server (Forbidden): error when creating "test-pvc2.yml": persistentvolumeclaims "test-pvc2" is forbidden: maximum storage usage per PersistentVolumeClaim is 2Gi, but request is 3Gi
```