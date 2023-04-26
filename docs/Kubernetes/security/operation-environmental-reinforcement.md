# 运行环境加固

## 最小特权原则(POLP)

`介绍`

```text
最小特权原则 (Principle of least privilege，POLP) ：是一种信息安全念，即为用户提供执行其工作职责所需的最小权限等级或许可。

最小特权原则被广泛认为是网络安全的最佳实践，也是保护高价值数据和资产的特权访问的基本方式。
```

`重要性`

最小特权原则 (POLP) 重要性：

- **减少网络攻击面**：当今，大多数高级攻击都依赖于利用特权凭证。通过限制超级用户和管理员权限，最小权限执行有助于减少总体网络攻击面。
- **阻止恶意软件的传播**： 通过在服务器或者在应用系统上执行最小权限，恶意软件攻击（例如SQL注入攻击）将很难提权来增加访问权限并横向移动破坏其他软件、设备。
- **有助于简化合规性和审核**：许多内部政策和法规要求都要求组织对特权帐户实施最小权限原则，以防止对关键业务系统的恶意破坏。最小权限执行可以帮助组织证明对特权活动的完整审核跟踪的合规性。

`实施`

在团队中实施最小特权原则 (POLP) ：

- 在所有服务器、业务系统中，审核整个环境以查找特权帐户（例如SSH账号、管理后台账号、跳板机账号）；
- 减少不必要的管理员权限，并确保所有用户和工具执行工作时所需的权限；
- 定期更改管理员账号密码；
- 监控管理员账号操作行为，告警通知异常活动。

## AppArmor限制容器对资源访问

`介绍`

```text
AppArmor（Application Armor） 是一个 Linux 内核安全模块，可用于限制主机操作系统上运行的进程的功能。每个进程都可以拥有自己的安全配置文件。安全配置文件用来允许或禁止特定功能，例如网络访问、文件读/写/执行权限等。

Linux发行版内置：Ubuntu、Debian
```

`工作模式`

Apparmor两种工作模式：

- **Enforcement（强制模式）**：在这种模式下，配置文件里列出的限制条件都会得到执行，并且对于违反这些限制条件的程序会进行日志记录。
- **Complain（投诉模式）**：在这种模式下，配置文件里的限制条件不会得到执行，Apparmor只是对程序的行为进行记录。一般用于调试。

`常用命令`

- apparmor_status：查看AppArmor配置文件的当前状态的
- apparmor_parser：将AppArmor配置文件加载到内核中
  - apparmor_parser <profile># 加载到内核中
  - apparmor_parser -r <profile># 重新加载配置
  - apparmor_parser -R <profile># 删除配置
- aa-complain：将AppArmor配置文件设置为投诉模式，需要安apparmor-utils软件包
- aa-enforce：将AppArmor配置文件设置为强制模式，需要安装apparmor-utils软件包

`k8s使用AppArmor的先决条件`

- K8s版本v1.4+，检查是否支持：kubectl describe node |grep AppArmor
- Linux内核已启用AppArmor，查看 cat /sys/module/apparmor/parameters/enabled
- 容器运行时需要支持AppArmor，像 Docker、CRI-O 或 containerd 都支持

**PS: AppArmor 目前处于测试阶段，因此在注解中指定AppArmor策略配置文件**

`配置示例`

```yml
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
  annotations:
    # 告知 Kubernetes 去应用 AppArmor 配置 "k8s-apparmor-example-deny-write"。
    # 请注意，如果节点上运行的 Kubernetes 不是 1.4 或更高版本，此注解将被忽略。
    # container.apparmor.security.beta.kubernetes.io/<container_name>: localhost/<profile_ref>
    # <container_name> Pod中容器名称
    # <profile_ref> Pod所在宿主机上策略名，默认目录/etc/apparmor.d
    container.apparmor.security.beta.kubernetes.io/hello: localhost/k8s-apparmor-example-deny-write
spec:
  containers:
  - name: hello
    image: busybox:1.28
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
```

`使用案例`

案例：容器文件系统访问限制

```text
步骤：
1、将自定义策略配置文件保存到/etc/apparmor.d/
2、加载配置文件到内核：apparmor_parser <profile>
3、Pod注解指定策略配置名
```

访问文件模式

| 字符 | 描述 |
| --- | --- |
| r | 读 |
| w | 写 |
| a | 追加 |
| k | 文件锁定 |
| l | 链接 |
| x | 可执行 |

匹配目录和文件

- 通配符 * : 在目录级别匹配零个或多个字符
  - /dir/* 匹配目录中的任何文件
  - /dir/a* 匹配目录中以a开头的任意文件
  - /dir/*.png 匹配目录中以.png结尾的任意文件
  - /dir/a*/ 匹配/dir里面以a开头的目录
  - /dir/*a/ 匹配/dir里面以a结尾的目录
- 通配符 ** : 在多个目录级别匹配零个或多个字符
  - /dir/** 匹配/dir目录或者/dir目录下任何文件和目录
  - /dir/**/ 匹配/dir或者/dir下面任何目录
- 通配符 [] [^] : 字符串，匹配其中任意字符
  - /dir/[^.]* 匹配/dir目录中以.之外的任何文件
  - /dir/**[^/] 匹配/dir目录或者/dir下面的任何目录中的任何文件

配置文件书写格式

- 第一行：导入依赖，遵循C语言约定
- 第二行：指定策略名
- 第三行：{} 策略块

```bash
cd /etc/apparmor.d
cat > k8s-example-deny-write <<EOF
#include <tunables/global>
profile k8s-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file, # 允许所有文件读写

  # 拒绝所有文件写入
  deny /tmp/** w,
  deny /data/www/** w,
}
EOF
# 加载到内核里
sudo apparmor_parser -r k8s-example-deny-write
# 查看状态
sudo apparmor_status | grep write
   k8s-example-deny-write
sudo cat ~/apparmor/test-apparmor-pod.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
  annotations:
    container.apparmor.security.beta.kubernetes.io/hello: localhost/k8s-example-deny-write
spec:
  containers:
  - name: hello
    image: busybox:1.28
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
EOF

kubectl apply -f ~/apparmor/test-apparmor-pod.yml
kubectl get po
NAME             READY   STATUS    RESTARTS   AGE
hello-apparmor   1/1     Running   0          4s
kubectl exec hello-apparmor -it -- sh
/ # cd /data/www/
/data/www # touch a
touch: a: Permission denied
/data/www # cd /tmp/
/tmp # touch a
touch: a: Permission denied
/tmp # cd /data/
/data # touch a
# 规则生效
```

![apparmor.png](https://s2.loli.net/2023/04/25/JtGnHeRzkdyAh19.png)

## Seccomp限制容器进程系统调用

`介绍`

```text
对于 Linux 来说，用户层一切资源相关操作都需要通过系统调用来完成；系统调用实现技术层次上解耦，内核只关心系统调用API的实现，而不必关心谁调用的。

Seccomp（Secure computing mode） 是一个 Linux 内核安全模块，可用于应用进程允许使用的系统调用。

容器实际上是宿主机上运行的一个进程，共享宿主机内核，如果所有容器都具有任何系统调用的能力，那么容器如果被入侵，就很轻松绕过容器隔离更改宿主机系统权限或者进入宿主机。

这就可以使用Seccomp机制限制容器系统调用，有效减少攻击面。

Linux发行版内置：CentOS、Ubuntu
```

![seccomp.png](https://s2.loli.net/2023/04/26/DK9csWCJflmTYiH.png)

`k8s版本使用区别和使用结构`

Seccomp在Kubernetes 1.3版本引入，在1.19版本成为GA版本，因此K8s中使Seccomp可以通过以下两种方式：

- 1.19版本之前

```yml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    seccomp.security.alpha.kubernetes.io/pod: "localhost/<profile>"
  name: audit-pod
  labels:
    app: audit-pod
spec:
  containers:
  - name: test-container
    image: hashicorp/http-echo:0.2.3
    args:
    - "-text=just made some syscalls!"
    securityContext:
      allowPrivilegeEscalation: false

```

- 1.19版本+

```yml
apiVersion: v1
kind: Pod
metadata:
  name: audit-pod
  labels:
    app: audit-pod
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/audit.json
  containers:
  - name: test-container
    image: hashicorp/http-echo:0.2.3
    args:
    - "-text=just made some syscalls!"
    securityContext:
      allowPrivilegeEscalation: false
```

- seccomp基本配置文件结构

  - defaultAction：在syscalls部分未定义的任何系统调用默认动作为允许
  - architectures: 架构
  - syscalls
    - names 系统调用名称，可以换行写多个
    - SCMP_ACT_ERRNO 阻止系统调用

```bash
sudo cat > ./fine-grained.json <<EOF
{
    "defaultAction": "SCMP_ACT_ALLOW",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
    ],
    "syscalls": [
        {
            "names": [
                "chmod",
                "mkdir"
            ],
            "action": "SCMP_ACT_ERRNO"
        }
    ]
}
EOF
```

- seccomp默认工作目录: **/var/lib/kubelet/seccomp/**

`使用示例: 禁止容器使用chmod`

```bash
# 让pod分配在指定节点,这里是n1,已经配置好seccomp文件
cat seccomp-test-pod.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-test-pod
  labels:
    test: seccomp-test-pod
spec:
  nodeName: n1
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: fine-grained.json
  containers:
  - name: test-container-nginx
    image: nginx
EOF
k apply -f seccomp-test-pod.yml
k get po
NAME               READY   STATUS   RESTARTS   AGE
seccomp-test-pod   0/1     Error    0          6s
k logs seccomp-test-pod
...
/docker-entrypoint.sh: Configuration complete; ready for start up
2023/04/26 04:19:11 [emerg] 1#1: mkdir() "/var/cache/nginx/client_temp" failed (1: Operation not permitted)
nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (1: Operation not permitted)
```

`启用使用 RuntimeDefault 作为所有工作负载的默认 seccomp 配置文件`

```text
大多数容器运行时都提供一组允许或不允许的默认系统调用。通过使用 runtime/default 注释 或将Pod 或容器的安全上下文中的 seccomp 类型设置为 RuntimeDefault，可以轻松地在 Kubernetes 中应用默认值。
```

<https://kubernetes.io/zh-cn/docs/tutorials/security/seccomp/#enable-runtimedefault-as-default>
