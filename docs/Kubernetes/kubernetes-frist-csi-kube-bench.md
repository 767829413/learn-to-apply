# CIS安全基准初探&&Kubernetes安全基准工具kube-bench

## CIS安全基准

```text
互联网安全中心（CIS，Center for Internet Security），是一个非盈利组织，致力为   互联网提供
免费的安全防御解决方案。
```
    
官网：<https://www.cisecurity.org/>

Kubernetes CIS基准：<https://www.cisecurity.org/benchmark/kubernetes/>

## K8s安全基准工具 kube-bench

```text
下载pdf后，根据里面的基准来检查K8s集群配置，但内容量太大，一般会采用相关工具来完成这项工作。
Kube-bench是容器安全厂商Aquq推出的工具，以CIS K8s基准作为基础，来检查K8s是否安全部署。
主要查找不安全的配置参数、敏感的文件权限、不安全的帐户或公开端口等等。
```

项目地址：<https://github.com/aquasecurity/kube-bench>

`部署`

```bash
wget https://github.com/aquasecurity/kube-bench/releases/download/v0.6.12/kube-bench_0.6.12_linux_amd64.tar.gz
tar zxvf kube-bench_0.6.12_linux_amd64.tar.gz
# 创建默认配置文件路径
mkdir /etc/kube-bench
mv cfg /etc/kube-bench/cfg
mv ./kube-bench /usr/bin/
```

`常用参数`

使用kube-bench run进行测试，该指令有以下常用参数：
常用参数：

- -s, --targets 指定要基础测试的目标，这个目标需要匹配cfg/<version>中的
文件名称，已有目标：master, controlplane, node, etcd, policies
- --version：指定k8s版本，如果未指定会自动检测
- --benchmark：手动指定CIS基准版本，不能与--version一起使用
- --json：将结果作为json内容输出

CIS Kubernetes Benchmark support: <https://github.com/aquasecurity/kube-bench/blob/main/docs/platforms.md#cis-kubernetes-benchmark-support>

`执行方式`

例如：检查master组件安全配置

执行后会逐个检查安全配置并输出修复方案及汇总信息输出：

- [PASS]：测试通过
- [FAIL]：测试未通过，重点关注，在测试结果会给出修复建议
- [WARN]：警告，可做了解
- [INFO]：信息

```bash
kube-bench run -s master
[INFO] 1 Control Plane Security Configuration
[INFO] 1.1 Control Plane Node Configuration Files
[PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 644 or more restrictive (Automated)
[PASS] 1.1.2 Ensure that the API server pod specification file ownership is set to root:root (Automated)
[PASS] 1.1.3 Ensure that the controller manager pod specification file permissions are set to 600 or more restrictive (Automated)
[PASS] 1.1.4 Ensure that the controller manager pod specification file ownership is set to root:root (Automated)
[PASS] 1.1.5 Ensure that the scheduler pod specification file permissions are set to 600 or more restrictive (Automated)
[PASS] 1.1.6 Ensure that the scheduler pod specification file ownership is set to root:root (Automated)
[FAIL] 1.1.7 Ensure that the etcd pod specification file permissions are set to 600 or more restrictive (Automated)
[FAIL] 1.1.8 Ensure that the etcd pod specification file ownership is set to root:root (Automated)
...

== Remediations master ==
1.1.7 Run the below command (based on the file location on your system) on the control plane node.
For example,
chmod 600 /usr/lib/systemd/system/etcd.service

1.1.8 Run the below command (based on the file location on your system) on the control plane node.
For example,
chown root:root /usr/lib/systemd/system/etcd.service

...

== Summary master ==
38 checks PASS
11 checks FAIL
12 checks WARN
0 checks INFO

== Summary total ==
38 checks PASS
11 checks FAIL
12 checks WARN
0 checks INFO
```

`配置文件`

```bash
# 默认目录是在: /etc/kube-bench/cfg
ls /etc/kube-bench/cfg
ack-1.0  cis-1.20  cis-1.24  cis-1.6      config.yaml  eks-1.1.0                 gke-1.0    rh-0.7
aks-1.0  cis-1.23  cis-1.5   cis-1.6-k3s  eks-1.0.1    eks-stig-kubernetes-v1r6  gke-1.2.0  rh-1.0
cat /etc/kube-bench/cfg/cis-1.24
---
controls:
version: "cis-1.24"
id: 1
text: "Control Plane Security Configuration"
type: "master"
groups:
...
      - id: 1.2.21
        text: "Ensure that the --audit-log-maxsize argument is set to 100 or as appropriate (Automated)"
        audit: "/bin/ps -ef | grep $apiserverbin | grep -v grep"
        tests:
          test_items:
            - flag: "--audit-log-maxsize"
              compare:
                op: gte
                value: 100
        remediation: |
          Edit the API server pod specification file $apiserverconf
          on the control plane node and set the --audit-log-maxsize parameter to an appropriate size in MB.
          For example, to set it as 100 MB, --audit-log-maxsize=100
        scored: true

...
```

字段解释:

- id：编号
- text：提示的文本
- audit：shell命令设置
- tests：测试项目
- remediation：修复方案
- scored：如果为true，kube-bench无法正常测试，则会生成FAIL，如果为false，无法正常测试，则会生成WARN。
- type：如果为manual则会生成WARN，如果为skip，
则会生成INFO
