# 监控,审计和运行时安全

## 分析容器系统调用：Sysdig

`介绍`

```text
Sysdig：一个非常强大的系统监控、分析和故障排查工具。

汇聚 strace+tcpdump+htop+iftop+lsof 工具功能于一身！

sysdig 除了能获取系统资源利用率、进程、网络连接、系统调用等信息，还具备了很强的分析能力，例如：

• 按照CPU使用率对进程排序
• 按照数据包对进程排序
• 打开最多的文件描述符进程
• 查看进程打开了哪些文件
• 查看进程的HTTP请求报文
• 查看机器上容器列表及资源使用情况

sysdig 通过在内核的驱动模块注册系统调用的 hook，这样当有系统调用发生和完成的时候，它会把系统调用信息拷贝到特定的buffer，然后用户态组件对数据信息处理（解压、解析、过滤等），并最终通过 sysdig 命令行和用户进行交互。
```

**项目地址：**<https://github.com/draios/sysdig>

**文档：**<https://github.com/draios/sysdig/wiki>

![sysdig-flow.png](https://s2.loli.net/2023/05/06/7wKGMprF4TI3qXR.png)

`安装`

**ubuntu-20.04 LTS 安装借鉴:** <https://www.linuxcapable.com/how-to-install-sysdig-on-ubuntu-20-04/>

```bash
# centos7.6 安装
rpm --import https://s3.amazonaws.com/download.draios.com/DRAIOS-GPG-KEY.public

curl -s -o /etc/yum.repos.d/draios.repo https://s3.amazonaws.com/download.draios.com/stable/rpm/draios.repo

yum install epel-release -y
yum install sysdig -y
/usr/bin/sysdig-probe-loader # 加载驱动模块
```

`常用参数`

* -l, --list：列出可用于过滤和输出的字段
* -M <num_seconds> ：多少秒后停止收集
* -p <output_format>, --print=<output_format> ：指定打印事件时使用的格式
  * 使用-pc或-pcontainer 容器友好的格式
  * 使用-pk或-pkubernetes k8s友好的格式
* -c <chiselname> <chiselargs>：指定内置工具，可直接完成具体的数据聚合、分析工作
* -w <filename>：保存到文件中
* -r <filename>：从文件中读取

`输出格式说明`

执行sysdig命令，实时输出大量系统调用。

示例：10919810 12:37:33.085981210 1 calico-node (2129.2151) < epoll_pwait

格式：%evt.num %evt.outputtime %evt.cpu %proc.name (%thread.tid) %evt.dir %evt.type %evt.info

* evt.num： 递增的事件号
* evt.time： 事件发生的时间
* evt.cpu： 事件被捕获时所在的 CPU，也就是系统调用是在哪个 CPU 执行的
* proc.name： 生成事件的进程名字
* thread.tid： 线程的 id，如果是单线程的程序，这也是进程的 pid
* evt.dir： 事件的方向（direction），> 代表进入事件，< 代表退出事件
* evt.type： 事件的名称，比如 open、stat等，一般是系统调用
* evt.args： 事件的参数。如果是系统调用，这些对应着系统调用的参数

自定义格式输出：sudo sysdig -p "user:%user.name time:%evt.time proc_name:%proc.name"

`过滤使用`

**sysdig过滤：**

* fd：根据文件描述符过滤，比如 fd 标号（fd.num）、fd 名字（fd.name）
* process：根据进程信息过滤，比如进程 id（proc.id）、进程名（proc.name）
* evt：根据事件信息过滤，比如事件编号、事件名
* user：根据用户信息过滤，比如用户 id、用户名、用户 home 目录
* syslog：根据系统日志过滤，比如日志的严重程度、日志的内容
* container：根据容器信息过滤，比如容器ID、容器名称、容器镜像

*查看完整过滤器列表：sudo sysdig -l*

```bash
# 示例：
# 查看一个进程的系统调用
sudo sysdig proc.name=kubelet
# 查看建立TCP连接的事件
sudo sysdig evt.type=accept
# 查看/etc目录下打开的文件描述符
sudo sysdig fd.name contains /etc
# 查看容器的系统调用
sudo sysdig -M 10 container.name=web
```

**注：还支持运算操作符，=、!=、>=、>、<、<=、contains、in 、exists、and、or、not**

`工具集成`

*Chisels：实用的工具箱，一组预定义的功能集合，用来分析特定的场景。*

**sysdig -cl 列出所有Chisels，以下是一些常用的：**

* sudo sysdig -c topprocs_cpu：输出按照 CPU 使用率排序的进程列表，例如sysdig -c
* sudo sysdig -c topprocs_net：输出进程使用网络TOP
* sudo sysdig -c topprocs_file：进程读写磁盘文件TOP
* sudo sysdig -c topfiles_bytes：读写磁盘文件TOP
* sudo sysdig -c netstat：列出网络的连接情况

**使用示例**

```bash
# 网络
# 查看使用网络的进程TOP
sudo sysdig -c topprocs_net
# 查看建立连接的端口
sudo sysdig -c fdcount_by fd.sport "evt.type=accept" -M 10
# 查看建立连接的端口
sudo sysdig -c fdbytes_by fd.sport
# 查看建立连接的IP
sudo sysdig -c fdcount_by fd.cip "evt.type=accept" -M 10
# 查看建立连接的IP
sudo sysdig -c fdbytes_by fd.cip

# 硬盘
# 查看进程磁盘I/O读写
sudo sysdig -c topprocs_file
# 查看进程打开的文件描述符数量
sudo sysdig -c fdcount_by proc.name "fd.type=file" -M 10
# 查看读写磁盘文件
sudo sysdig -c topfiles_bytes
sudo sysdig -c topfiles_bytes proc.name=etcd
# 查看/tmp目录读写磁盘活动文件
sudo sysdig -c fdbytes_by fd.filename "fd.directory=/tmp/"

# CPU
# 查看CPU使用率TOP
sudo sysdig -c topprocs_cpu
# 查看容器CPU使用率TOP
sudo sysdig -pc -c topprocs_cpu container.name=web
sudo sysdig -pc -c topprocs_cpu container.id=web

# 容器
# 查看机器上容器列表及资源使用情况
sudo csysdig -vcontainers
# 查看容器资源使用TOP
sudo sysdig -c topcontainers_cpu/topcontainers_net/topcontainers_file

# 其他常用命令
sudo sysdig -c netstat
sudo sysdig -c ps
sudo sysdig -c lsof
```

## 监控容器运行时：Falco

`介绍`

```text
Falco 是一个 Linux 安全工具，它使用系统调用来保护和监控系统。

Falco最初是由Sysdig开发的，后来加入CNCF孵化器，成为首个加入CNCF的运行时安全项目。

Falco提供了一组默认规则，可以监控内核态的异常行为，例如：

• 对于系统目录/etc, /usr/bin, /usr/sbin的读写行为
• 文件所有权、访问权限的变更
• 从容器打开shell会话
• 容器生成新进程
• 特权容器启动
```

**项目地址：**<https://github.com/falcosecurity/falco>

![falco-架构.png](https://s2.loli.net/2023/05/06/IzuPLer6AdgbcU8.png)

`安装`

<https://falco.org/docs/getting-started/installation/>

**这里以 Ubuntu 举例**

```bash
# Trust the falcosecurity GPG key
curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg

# Configure the apt repository
echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main" | sudo tee -a /etc/apt/sources.list.d/falcosecurity.list

# Update the package list
sudo apt-get update -y

# Install some required dependencies that are needed to build the kernel module and the BPF probe
sudo apt install -y dkms make linux-headers-$(uname -r)
# If you use the falco-driver-loader to build the BPF probe locally you need also clang toolchain
sudo apt install -y clang llvm
# You can install also the dialog package if you want it
sudo apt install -y dialog

# Install the Falco package
sudo apt-get install -y falco
```

`配置`

**配置文件目录：/etc/falco**

```bash
sudo ls /etc/falco -l
total 224
-rw-r--r-- 1 root root  12314 Jan 17 20:11 aws_cloudtrail_rules.yaml
-rw-r--r-- 1 root root     21 Feb 20 19:01 falco_rules.local.yaml
-rw-r--r-- 1 root root 147885 Jan  1  1970 falco_rules.yaml
-rw-r--r-- 1 root root  19183 Feb 20 18:59 falco.yaml
-rw-r--r-- 1 root root  31009 Jan 17 20:13 k8s_audit_rules.yaml
drwxr-xr-x 2 root root   4096 Feb 20 19:23 rules.d
```

* falco.yaml falco配置与输出告警通知方式
* falco_rules.yaml 规则文件，默认已经定义很多威胁场景
* falco_rules.local.yaml 自定义扩展规则文件
* k8s_audit_rules.yaml K8s审计日志规则

**告警规则示例（falco_rules.local.yaml）：**

```yaml
- rule: The program "sudo" is run in a container
  desc: An event will trigger every time you run sudo in a container
  condition: evt.type = execve and evt.dir=< and container.id != host and proc.name = sudo
  output: "Sudo run in container (user=%user.name %container.info parent=%proc.pname cmdline=%proc.cmdline)"
  priority: ERROR
  tags: [users, container]
```

**参数说明**

* rule：规则名称，唯一
* desc：规则的描述
* condition： 条件表达式
* output：符合条件事件的输出格式
* priority：告警的优先级
* tags：本条规则的 tags 分类

`使用示例`

**威胁场景测试：**

*默认规则可以查看这个官方文档:* <https://github.com/falcosecurity/rules/blob/main/rules_inventory/rules_overview.md#falco-rules---summary-stats>

```bash
# 监控系统二进制文件目录读写（默认规则）
# 在当前窗口执行
sudo echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
sudo touch /usr/bin/aaaa && sudo rm  -f /usr/bin/aaaa
# 另一个窗口输出
sudo tail -f /var/log/syslog | grep falco
May  6 15:29:21 m1 falco: 15:29:21.212161973: Error File below a known binary directory opened for writing (user=root user_loginuid=1000 command=touch /usr/bin/aaaa pid=145225 file=/usr/bin/aaaa parent=sudo pcmdline=sudo touch /usr/bin/aaaa gparent=bash container_id=host image=<NA>)
May  6 15:29:21 m1 falco: 15:29:21.216553426: Error File below known binary directory renamed/removed (user=root user_loginuid=1000 command=rm -f /usr/bin/aaaa pid=145227 pcmdline=sudo rm -f /usr/bin/aaaa operation=unlinkat file=<NA> res=0 dirfd=-100(AT_FDCWD) name=/usr/bin/aaaa flags=0  container_id=host image=<NA>)

# 监控根目录或者/root目录写入文件（默认规则）
# 监控运行交互式Shell的容器（默认规则）
```

**监控容器创建的不可信任进程（自定义规则）**

```bash
sudo cat > /etc/falco/falco_rules.local.yaml <<EOF
- rule: Unauthorized process on nginx containers
  condition: spawned_process and container and container.image startswith nginx and not proc.name in (nginx)
  desc: test
  output: "Unauthorized process on nginx containers (user=%user.name container_name=%container.name container_id=%container.id image=%container.image.repository shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline terminal=%proc.tty)"
  priority: WARNING
EOF

# 重启falco应用新配置文件
sudo systemctl restart falco

# 启动一个nginx容器,进入容器里执行 tail -f /var/log/nginx/access.log
```

**condition表达式解读：**

* spawned_process 运行新进程
* container 容器
* container.image startswith nginx 以nginx开头的容器镜像
* not proc.name in (nginx) 不属于nginx的进程名称（允许进程名称列表）

验证：tail -f /var/log/syslog | grep falco（告警通知默认输出到标准输出和系统日志）

`Falco支持五种输出告警通知的方式`

**告警配置文件：/etc/falco/falco.yaml**

```yaml
# 输出格式为json,默认false
json_output: false

# 输出到标准输出（默认启用）
stdout_output:
  enabled: true

# 输出到Syslog（默认启用）
syslog_output:
  enabled: true

# 输出到文件
file_output:
  enabled: false
  keep_alive: false # true 打开一次文件持续写入, false 有输出就打开文件写入
  filename: ./events.txt

# 输出到其他程序（命令行管道方式）
program_output:
  enabled: false
  keep_alive: false
  program: "jq '{text: .output}' | curl -d @- -X POST https://hooks.slack.com/services/XXX"

# 输出到HTTP服务
http_output:
  enabled: false
  url: http://some.url
  user_agent: "falcosecurity/falco"

# grpc输出
grpc_output:
  enabled: false
```

`可视化监控`

**Falco告警集中化展示**

![falco-gui.png](https://s2.loli.net/2023/05/08/sC8WXfaUwMN6uy2.png)

* FalcoSideKick：一个集中收集并指定输出，支持大量方式输出，例如Influxdb、Elasticsearch等
  * 项目地址 <https://github.com/falcosecurity/falcosidekick>
* FalcoSideKick-UI：告警通知集中图形展示系统
  * 项目地址 <https://github.com/falcosecurity/falcosidekick-ui>

**docker搭建**

1. *配置http输出*

```yaml
json_output: true
json_include_output_property: true
http_output:
  enabled: true
  url: http://192.168.148.119:2801
  user_agent: "falcosecurity/falco"
```

2. *docker部署http服务器接收*

*项目地址:*

<https://github.com/falcosecurity/falcosidekick>

<https://github.com/falcosecurity/falcosidekick-ui>

```bash
docker run -d \
-p 2801:2801 \
--name falcosidekick \
-e WEBUI_URL=http://192.168.148.119:2802 \
falcosecurity/falcosidekick

docker run -d -p 6379:6379 redislabs/redisearch:2.2.4

docker ps | grep redis
e77497e1a468   redislabs/redisearch:2.2.4    "docker-entrypoint.s…"   23 minutes ago   Up 23 minutes   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp   admiring_goldberg

docker inspect admiring_goldberg | grep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.3",
                    "IPAddress": "172.17.0.3",
# 账号密码 <login>:<password> => admin:admin
docker run -d \
-p 2802:2802 \
-e FALCOSIDEKICK_UI_REDIS_URL=172.17.0.3:6379 \
-e FALCOSIDEKICK_UI_USER=admin:admin \
--name falcosidekick-ui \
falcosecurity/falcosidekick-ui 
```

*UI访问地址：<http://192.168.148.119:2802/login>*

## Kubernetes 审计日志

`介绍`

```text
在Kubernetes集群中，API Server的审计日志记录了哪些用户、哪些服务请求操作集群资源，并且可以编写不同规则，控制忽略、存储的操作日志。

审计日志采用JSON格式输出，每条日志都包含丰富的元数据，例如请求的URL、HTTP方法、客户端来源等，你可以使用监控服务来分析API流量，以检测趋势或可能存在的安全隐患。
```

**这些可能服务会访问API Server：**

* 管理节点（controller-manager、scheduler）
* 工作节点（kubelet、kube-proxy）
* 集群服务（CoreDNS、Calico、HPA等）
* kubectl、API、Dashboard

`事件&&阶段`

*当客户端向 API Server发出请求时，该请求将经历一个或多个阶段：*

| 阶段 | 说明 | 
| :-----: | :----: |
| RequestReceived | 审核处理程序已收到请求 |
| ResponseStarted | 已发送响应标头，但尚未发送响应正文 |
| ResponseComplete | 响应正文已完成，不再发送任何字节 |
| Panic | 内部服务器出错，请求未完成 |

![api-server-rr-flow.png](https://s2.loli.net/2023/05/09/SVuqwFjQvgLpAZX.png)

`配置&&级别`

*Kubernetes审核策略文件包含一系列规则，描述了记录日志的级别，采集哪些日志，不采集哪些日志。*

| 级别 | 说明 | 
| :-----: | :----: |
| None | 不为事件创建日志条目 |
| Metadata | 创建日志条目。包括元数据，但不包括请求正文或响应正文 |
| Request | 创建日志条目。包括元数据和请求正文，但不包括响应正文 |
| RequestResponse | 创建日志条目。包括元数据、请求正文和响应正文|

**审计策略配置格式**

```yaml
apiVersion: audit.k8s.io/v1 # 这是必填项。
kind: Policy
# 不要在 RequestReceived 阶段为任何请求生成审计事件。
omitStages:
  - "RequestReceived"
rules:
  # 在日志中用 RequestResponse 级别记录 Pod 变化。
  - level: RequestResponse
    resources:
    - group: ""
      # 资源 "pods" 不匹配对任何 Pod 子资源的请求，
      # 这与 RBAC 策略一致。
      resources: ["pods"]
  # 在日志中按 Metadata 级别记录 "pods/log"、"pods/status" 请求
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # 不要在日志中记录对名为 "controller-leader" 的 configmap 的请求。
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # 不要在日志中记录由 "system:kube-proxy" 发出的对端点或服务的监测请求。
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API 组
      resources: ["endpoints", "services"]

  # 不要在日志中记录对某些非资源 URL 路径的已认证请求。
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # 通配符匹配。
    - "/version"

  # 在日志中记录 kube-system 中 configmap 变更的请求消息体。
  - level: Request
    resources:
    - group: "" # core API 组
      resources: ["configmaps"]
    # 这个规则仅适用于 "kube-system" 名字空间中的资源。
    # 空字符串 "" 可用于选择非名字空间作用域的资源。
    namespaces: ["kube-system"]

  # 在日志中用 Metadata 级别记录所有其他名字空间中的 configmap 和 secret 变更。
  - level: Metadata
    resources:
    - group: "" # core API 组
      resources: ["secrets", "configmaps"]

  # 在日志中以 Request 级别记录所有其他 core 和 extensions 组中的资源操作。
  - level: Request
    resources:
    - group: "" # core API 组
    - group: "extensions" # 不应包括在内的组版本。

  # 一个抓取所有的规则，将在日志中以 Metadata 级别记录所有其他请求。
  - level: Metadata
    # 符合此规则的 watch 等长时间运行的请求将不会
    # 在 RequestReceived 阶段生成审计事件。
    omitStages:
      - "RequestReceived"
```

**参考资料：**<https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/audit/>

`开启和示例使用`

*打开 /etc/kubernetes/manifests/kube-apiserver.yaml*

```bash
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
...
spec:
  containers:
  - command:
    - kube-apiserver
...
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml # 审计日志策略文件
    - --audit-log-path=/var/log/kubernetes/audit/audit.log # 指定用来写入审计事件的日志文件路径。不指定此标志会禁用日志后端。- 意味着标准化
    - --audit-log-maxage=30 # 定义保留旧审计日志文件的最大天数
    - --audit-log-maxbackup=10 # 定义要保留的审计日志文件的最大数量
    - --audit-log-maxsize=100 # 定义审计日志文件轮转之前的最大大小（兆字节）
    - --audit-webhook-config-file=/etc/kubernetes/webhook-config.yaml 设置 Webhook 配置文件的路径。Webhook 配置文件实际上是一个 kubeconfig 文件
    - --audit-webhook-initial-backoff=10 指定在第一次失败后重发请求等待的时间。随后的请求将以指数退避重试
...
# 如果你的集群控制面以 Pod 的形式运行 kube-apiserver，记得要通过 hostPath 卷来访问策略文件和日志文件所在的目录，这样审计记录才会持久保存下来。

# 挂载数据卷
...
volumeMounts:
  - mountPath: /etc/kubernetes/audit-policy.yaml
    name: audit
    readOnly: true
  - mountPath: /var/log/kubernetes/audit/
    name: audit-log
    readOnly: false
  - mountPath: /etc/kubernetes/webhook-config.yaml
    name: webhook
    readOnly: true
...

# 配置 hostPath
...
volumes:
- name: audit
  hostPath:
    path: /etc/kubernetes/audit-policy.yaml
    type: File

- name: audit-log
  hostPath:
    path: /var/log/kubernetes/audit/
    type: DirectoryOrCreate

- name: webhook
  hostPath:
    path: /etc/kubernetes/webhook-config.yaml
    type: File
...

# 可以测试下
sudo tail -f /var/log/kubernetes/audit/audit.log
{"kind":"Event","apiVersion":"audit.k8s.io/v1","level":"Metadata","auditID":"f86d01a7-f096-4fa8-8e5b-3498f63971c3","stage":"ResponseComplete","requestURI":"/apis/discovery.k8s.io/v1/namespaces/default/endpointslices/kubernetes","verb":"get","user":{"username":"system:apiserver","uid":"30e02858-1b1d-4ab9-ba76-b0b1b0814048","groups":["system:masters"]},"sourceIPs":["::1"],"userAgent":"kube-apiserver/v1.27.0 (linux/amd64) kubernetes/1b4df30","objectRef":{"resource":"endpointslices","namespace":"default","name":"kubernetes","apiGroup":"discovery.k8s.io","apiVersion":"v1"},"responseStatus":{"metadata":{},"code":200},"requestReceivedTimestamp":"2023-05-09T02:26:50.232676Z","stageTimestamp":"2023-05-09T02:26:50.233242Z","annotations":{"authorization.k8s.io/decision":"allow","authorization.k8s.io/reason":""}}
...
```

`示例: 只记录指定资源操作日志`

```bash
sudo cat > /etc/kubernetes/my-audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1
kind: Policy
# 忽略步骤，不为RequestReceived阶段生成审计日志
omitStages:
- "RequestReceived"
rules:
# 不记录日志
- level: None
  users:
  - system:apiserver
  - system:kube-controller-manager
  - system:kube-scheduler
  - system:kube-proxy
  - kubelet
# 针对资源记录日志
- level: Metadata
  resources:
  - group: ""
    resources: ["pods"]
  # - group: "apps"
  #   resources: ["deployments"]
# 其他资源不记录日志
- level: None
EOF

sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
...
spec:
  containers:
  - command:
    - kube-apiserver
...
    - --audit-policy-file=/etc/kubernetes/my-audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/my-audit.log
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
...
volumeMounts:
  - mountPath: /etc/kubernetes/my-audit-policy.yaml
    name: audit
    readOnly: true
  - mountPath: /var/log/kubernetes/audit/
    name: audit-log
    readOnly: false
...
volumes:
- name: audit
  hostPath:
    path: /etc/kubernetes/my-audit-policy.yaml
    type: File
- name: audit-log
  hostPath:
    path: /var/log/kubernetes/audit/
    type: DirectoryOrCreate
...


# 测试一下
kubectl get pod -A
sudo tail -f /var/log/kubernetes/audit/my-audit.log
```

`收集审计日志方案`

* 审计日志文件+filebeat
* 审计webhook+logstash
* 审计webhook+falco