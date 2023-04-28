# 最小化微服务漏洞

## Pod安全上下文

`介绍`

```text
安全上下文（Security Context）：K8s对Pod和容器提供的安全机制，可以设置Pod特权和访问控制。
```

`安全上下文限制维度`

- 自主访问控制（Discretionary Access Control）：基于用户ID（UID）和组ID（GID），来判定对对象（例如文件）的访问权限。
- 安全性增强的 Linux（SELinux）： 为对象赋予安全性标签。
- 以特权模式或者非特权模式运行。
- Linux Capabilities: 为进程赋予 root 用户的部分特权而非全部特权。
- AppArmor：定义Pod使用AppArmor限制容器对资源访问限制
- Seccomp：定义Pod使用Seccomp限制容器进程的系统调用
- AllowPrivilegeEscalation： 禁止容器中进程（通过 SetUID 或 SetGID 文件模式）获得特提升。当容器以特权模式运行或者具有CAP_SYS_ADMIN能力时，AllowPrivilegeEscalation总为True。
- readOnlyRootFilesystem：以只读方式加载容器的根文件系统。

`案例1：设置容器以普通用户运行`

```text
背景：容器中的应用程序默认以root账号运行的，这个root与宿主机root账号是相同的，拥有大部分对Linux内核的系统调用权限，这样是不安全的，所以我们应该将容器以普通用户运行，减少应用程序对权限的使用。
```

**可以通过两种方法设置普通用户：**

- Dockerfile里使用USER指定运行用户 

```Dockerfile
FROM python
# 添加用户
RUN useradd python
RUN mkdir /data/www -p
COPY . /data/www
# 为数据目录添加用户所属
RUN chown -R python /data
RUN pip install flask -i https://mirrors.aliyun.com/pypi/simple/ && \
    pip install prometheus_client -i https://mirrors.aliyun.com/pypi/simple/
WORKDIR /data/www
# 设置容器用户
USER python
CMD python main.py
```

- K8s里指定spec.securityContext.runAsUser，指定容器默认用户UID

**进行操作:**

```bash
cat > security-context-demo.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000 #  Pod 中的所有容器内的进程都使用用户 ID 1000 来运行,镜像里必须有这个用户UID
    runAsGroup: 3000 # 所有容器中的进程都以主组 ID 3000 来运行。 如果忽略此字段，则容器的主组 ID 将是 root（0）,当 runAsGroup 被设置时，所有创建的文件也会划归用户 1000 和组 3000
    fsGroup: 1000 # # 数据卷挂载后的目录属组设置为该组
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:
      allowPrivilegeEscalation: false # 不允许提权
EOF

kubectl apply -f security-context-demo.yml
kubectl get po
NAME                    READY   STATUS    RESTARTS   AGE
security-context-demo   1/1     Running   0          3s
kubectl exec -it security-context-demo -- sh
/ $ ps
PID   USER     TIME  COMMAND
    1 1000      0:00 sleep 1h
    6 1000      0:00 sh
   12 1000      0:00 ps
/ $ id
uid=1000 gid=3000 groups=1000
/ $ cd /data
/data $ ls -l
total 4
drwxrwsrwx    2 root     1000          4096 Apr 26 08:16 demo
```

`案例2：避免使用特权容器`

```text
背景：容器中有些应用程序可能需要访问宿主机设备、修改内核等需求，在默认情况下，容器没这个有这个能力，因此这时会考虑给容器设置特权模式
```

**启用特权模式:**

```yml
containers:
- image: lizhenliang/flask-demo:root
  name: web
  securityContext:
    privileged: true
```

*启用特权模式就意味着，你要为容器提供了访问Linux内核的所有能力，这是很危险的，为了减少系统调用的供给，可以使用Capabilities为容器赋予仅所需的能力。*

**Linux Capabilities：**

<https://man7.org/linux/man-pages/man7/capabilities.7.html>

```text
Capabilities 是一个内核级别的权限，它允许对内核调用权限进行更细粒度的控制，而不是简单地以 root 身份能力授权。

Capabilities 包括更改文件权限、控制网络子系统和执行系统管理等功能。在securityContext 中，可以添加或删除 Capabilities，做到容器精细化权限控制。
```

*PS: Linux 权能常数定义的形式为 CAP_XXX。但是你在 container 清单中列举权能时，要将权能名称中的 CAP_ 部分去掉。例如，要添加 CAP_SYS_TIME， 可在权能列表中添加 SYS_TIME*

**demo1: 容器默认没有挂载文件系统能力，添加SYS_ADMIN增加这个能力**

```bash
cat security-context-demo-1.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-1
spec:
  containers:
  - name: sec-ctx-1
    image: busybox
    command:
    - sleep
    - 24h
    securityContext:
      capabilities:
        add: ["SYS_ADMIN"]
EOF
kubectl apply -f security-context-demo-1.yml
kubectl get po
NAME                      READY   STATUS    RESTARTS   AGE
security-context-demo-1   1/1     Running   0          4s
kubectl exec -it security-context-demo-1 -- sh
mount -t tmpfs /tmp /tmp
# / # mount -t tmpfs /tmp /tmp
# mount: mounting /tmp on /tmp failed: Permission denied
```

**demo2: 只读挂载容器文件系统，防止恶意二进制文件创建**

```bash
cat security-context-demo-2.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-2
spec:
  containers:
  - name: sec-ctx-2
    image: busybox
    command:
    - sleep
    - 24h
    securityContext:
      readOnlyRootFilesystem: true
EOF
kubectl apply -f security-context-demo-2.yml
kubectl get po
NAME                      READY   STATUS    RESTARTS   AGE
security-context-demo-2   1/1     Running   0          4s
kubectl exec -it security-context-demo-2 -- sh
/ # touch a
touch: a: Read-only file system
```

## Pod 安全性标准（Pod Security Standards） &&　Pod 安全性标准（Security Standard）

`介绍`

[Pod 安全性标准](https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-standards/)

[Pod 安全性准入](https://kubernetes.io/zh-cn/docs/concepts/security/pod-security-admission/)

```text
Pod 安全性标准定义了三种不同的策略（Policy），以广泛覆盖安全应用场景。 这些策略是叠加式的（Cumulative），安全级别从高度宽松至高度受限。

Kubernetes Pod 安全性标准（Security Standard） 为 Pod 定义不同的隔离级别。这些标准能够让你以一种清晰、一致的方式定义如何限制 Pod 行为。

Kubernetes 提供了一个内置的 Pod Security 准入控制器来执行 Pod 安全标准 （Pod Security Standard）。 创建 Pod 时在名字空间级别应用这些 Pod 安全限制。
```

| Profile |	描述 |
| --- | --- |
| Privileged |	不受限制的策略，提供最大可能范围的权限许可。此策略允许已知的特权提升。|
| Baseline |	限制性最弱的策略，禁止已知的策略提升。允许使用默认的（规定最少）Pod 配置。|
| Restricted |	限制性非常强的策略，遵循当前的保护 Pod 的最佳实践 |

`为namespace设置 Pod 安全性准入控制标签`

| 模式 |	描述 |
| --- | --- |
| enforce | 策略违例会导致 Pod 被拒绝 |
| audit |	策略违例会触发审计日志中记录新事件时添加审计注解；但是 Pod 仍是被接受的。 |
| warn | 策略违例会触发用户可见的警告信息，但是 Pod 仍是被接受的。 |

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: my-baseline-namespace
  labels:
    # 模式的级别标签用来标示对应模式所应用的策略级别
    #
    # MODE 必须是 `enforce`、`audit` 或 `warn` 之一
    # LEVEL 必须是 `privileged`、baseline` 或 `restricted` 之一
    # pod-security.kubernetes.io/<MODE>: <LEVEL>
    pod-security.kubernetes.io/enforce: baseline
    # 可选：针对每个模式版本的版本标签可以将策略锁定到
    # 给定 Kubernetes 小版本号所附带的版本（例如 v1.27）
    #
    # MODE 必须是 `enforce`、`audit` 或 `warn` 之一
    # VERSION 必须是一个合法的 Kubernetes 小版本号或者 `latest`
    # pod-security.kubernetes.io/<MODE>-version: <VERSION>
    pod-security.kubernetes.io/enforce-version: v1.27

    # 我们将这些标签设置为我们所 _期望_ 的 `audit` `warn` 级别
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.27
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.27
```

`案例演示`

```bash
cat test-ns-1.yml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: test-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted 
    pod-security.kubernetes.io/enforce-version: v1.27
EOF
kubectl apply -f test-ns-1.yml

cat test-pod.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: test-ns
spec:
  containers:
  - name: test-pod
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]
    securityContext:
      allowPrivilegeEscalation: true
EOF
kubectl apply -f test-pod.yml
Error from server (Forbidden): error when creating "test-pod.yml": pods "test-pod" is forbidden: violates PodSecurity "restricted:v1.27": allowPrivilegeEscalation != false (container "test-pod" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "test-pod" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "test-pod" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "test-pod" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")

cat test-ns-2.yml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: test-ns
  labels:
    pod-security.kubernetes.io/enforce: baseline 
    pod-security.kubernetes.io/enforce-version: v1.27
EOF
k apply -f test-ns-2.yml
kubectl apply -f test-pod.yml
kubectl get pod -n test-ns
NAME       READY   STATUS    RESTARTS   AGE
test-pod   1/1     Running   0          16s
```

`使用 kubectl label 为现有 namespace 添加标签`

```bash
# 列举那些没有显式设置 enforce 级别的 namespace
kubectl get namespaces --selector='!pod-security.kubernetes.io/enforce'
# 应用到所有 namespace
kubectl label --overwrite ns --all \
  pod-security.kubernetes.io/audit=baseline \
  pod-security.kubernetes.io/warn=baseline
# 应用到单个 namespace
kubectl label --overwrite ns my-existing-namespace \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=v1.27
```

`从 PodSecurityPolicy 映射到 Pod 安全性标准`

<https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/psp-to-pod-security-standards/>

`从 PodSecurityPolicy 迁移到内置的 PodSecurity 准入控制器`

<https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/migrate-from-psp/>

## Secret存储敏感数据

`介绍`

<https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/#uses-for-secrets>

```text
Secret是一个用于存储敏感数据的资源，所有的数据要经过base64编码，数据实际会存储在K8s中Etcd，然后通过创建Pod时引用该数据。

应用场景：凭据
```

**Pod使用secret数据有两种方式：**

- 变量注入
- 数据卷挂载

**kubectl create secret 支持三种数据类型：**

- docker-registry：存储镜像仓库认证信息
- generic：从文件、目录或者字符串创建，例如存储用户名密码
- tls：存储证书，例如HTTPS证书

`示例：将Mysql用户密码保存到Secret中存储`

```bash
#　可以用命令行的方式创建
# kubectl create secret generic test-secret --from-literal='username=my-app' --from-literal='password=39528$vdg7Jb'
cat > secret-test.yml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
data:
  mysql-root-passwd: MTIzNDU2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    test: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      test: mysql
  template:
    metadata:
      labels:
        test: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7.30
        env: 
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-passwd
EOF


```

## 安全沙箱运行容器-gVisor

`介绍`

*项目地址：*<https://github.com/google/gvisor>

```text
容器的应用程序可以直接访问Linux内核的系统调用，容器在安全隔离上还是比较弱，虽然内核在不断地增强自身的安全特性，但由于内核自身代码极端复杂，CVE 漏洞层出不穷。

所以要想减少这方面安全风险，就是做好安全隔离，阻断容器内程序对物理机内核的依赖。

Google开源的一种gVisor容器沙箱技术就是采用这种思路，gVisor隔离容器内应用和内核之间访问，提供了大部分Linux内核的系统调用，巧妙的将容器内进程的系统调用转化为对gVisor的访问。

gVisor兼容OCI，与Docker和K8s无缝集成，很方面使用。
```

![gVisor.png](https://s2.loli.net/2023/04/28/3ueyzIovF5bNahD.png)

`gVisor架构`

**gVisor 由 3 个组件构成：**

- Runsc 是一种 Runtime 引擎，负责容器的创建与销毁。
- Sentry 负责容器内程序的系统调用处理。
- Gofer 负责文件系统的操作代理，IO 请求都会由它转接到 Host 上。

![gVisor-Architecture.png](https://s2.loli.net/2023/04/28/V8XemyqBWZru1Ox.png)

`集成gVisor`

**gVisor与 Docker 集成：**<https://gvisor.dev/docs/user_guide/install/>

**gVisor与 Containerd 集成：**<https://gvisor.dev/docs/user_guide/containerd/quick_start/>

**已经测试过的应用和工具：**<https://gvisor.dev/docs/user_guide/compatibility/>

`K8s使用gVisor运行容器`

<https://kubernetes.io/zh-cn/docs/concepts/containers/runtime-class/>

```text
RuntimeClass 是一个用于选择容器运行时配置的特性，容器运行时配置用
于运行 Pod 中的容器
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-gvisor
spec:
  runtimeClassName: gvisor
  containers:
  - name: nginx
    image: nginx
EOF

kubectl get pod nginx-gvisor -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP              NODE   NOMINATED NODE   READINESS GATES
nginx-gvisor   1/1     Running   0          34s   10.244.217.19   n2     <none>           <none>

kubectl exec nginx-gvisor -- dmesg
[    0.000000] Starting gVisor...
[    0.506550] Generating random numbers by fair dice roll...
[    0.807230] Synthesizing system calls...
[    1.148878] Searching for needles in stacks...
[    1.447969] Digging up root...
[    1.494686] Checking naughty and nice process list...
[    1.975122] Creating cloned children...
[    2.098729] Preparing for the zombie uprising...
[    2.591091] Feeding the init monster...
[    2.676737] Segmenting fault lines...
[    2.782574] Reading process obituaries...
[    2.929231] Setting up VFS...
[    2.985855] Setting up FUSE...
[    3.152612] Ready!
```

**PS: 如果遇到下述错误,建议删除对应节点的 calico pod**

```text
Events:
  Type     Reason                  Age                   From               Message
  ----     ------                  ----                  ----               -------
  Normal   Scheduled               25m                   default-scheduler  Successfully assigned default/nginx-gvisor to n2
  Warning  FailedCreatePodSandBox  25m                   kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "1479edf83beedb6d7797323c99cc413e02624fca5eb9e46840e775368737dc7e": plugin type="calico" failed (add): error getting ClusterInformation: connection is unauthorized: Unauthorized
```
