# Kind加载本地镜像失败

## 问题产生

使用 kind load 将镜像加载到本地k8s集群节点

```bash
kind load docker-image resouer/ubuntu-bc:latest
```

执行一个job的yaml

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
        - name: pi
          image: resouer/ubuntu-bc:latest  
          command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```

然后会出现镜像拉取失败

```text
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  15s   default-scheduler  Successfully assigned default/pi-njt9b to kind-control-plane
  Normal   Pulling    15s   kubelet            Pulling image "resouer/ubuntu-bc:latest"
  Warning  Failed     12s   kubelet            Failed to pull image "resouer/ubuntu-bc:latest": rpc error: code = Unknown desc = failed to pull and unpack image "docker.io/resouer/ubuntu-bc:latest": failed to resolve reference "docker.io/resouer/ubuntu-bc:latest": failed to do request: Head "https://registry-1.docker.io/v2/resouer/ubuntu-bc/manifests/latest": proxyconnect tcp: dial tcp 172.18.112.1:7890: connect: no route to host
  Warning  Failed     12s   kubelet            Error: ErrImagePull
  Normal   BackOff    11s   kubelet            Back-off pulling image "resouer/ubuntu-bc:latest"
  Warning  Failed     11s   kubelet            Error: ImagePullBackOff
```

## 解决思路

首先, `Kind` 是将 `k8s` 集群部署到本地的 `docker` 容器里的,每个集群只有一个节点

```bash
docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED      STATUS       PORTS                       NAMES
088a12ad7a0d   kindest/node:v1.24.0   "/usr/local/bin/entr…"   6 days ago   Up 2 hours   127.0.0.1:39217->6443/tcp   kind-control-plane
```

进入这个容器看下有没有可用的镜像

```bash
docker exec -it kind-control-plane bash
root@kind-control-plane:/# ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 03:33 ?        00:00:00 /sbin/init
root         212       1  0 03:33 ?        00:00:00 /lib/systemd/systemd-journald
root         225       1  0 03:33 ?        00:01:18 /usr/local/bin/containerd
root         260       1  2 03:33 ?        00:03:56 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/con
root         475       1  0 03:33 ?        00:00:02 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id f039f3c55a5258b61b048cdcb8587fd561eaf6b1151fe234703aa0464e0762be -address /run/containerd
root         496       1  0 03:33 ?        00:00:02 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id a4b939b6cc804b13154883525124e98b038f6de9e7b43772e6ad5da41c7349bf -address /run/containerd
root         508       1  0 03:33 ?        00:00:02 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id b82b57669d783d5c4e73a9db2499642f35987a532037cabfb3f10f4ff7d5022a -address /run/containerd
root         533       1  0 03:33 ?        00:00:02 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 553e1f81cd8d4ff2613ebcc1c5365911d87d1da623f2637debdec57b162e3998 -address /run/containerd
65535        578     475  0 03:33 ?        00:00:00 /pause
65535        585     496  0 03:33 ?        00:00:00 /pause
65535        593     508  0 03:33 ?        00:00:00 /pause
65535        600     533  0 03:33 ?        00:00:00 /pause
root         706     496  1 03:33 ?        00:02:44 etcd --advertise-client-urls=https://172.18.0.2:2379 --cert-file=/etc/kubernetes/pki/etcd/server.crt --client-cert-auth=true --data-dir=/var/lib/etcd 
root         714     475  1 03:33 ?        00:02:29 kube-scheduler --authentication-kubeconfig=/etc/kubernetes/scheduler.conf --authorization-kubeconfig=/etc/kubernetes/scheduler.conf --bind-address=127
root         722     533  3 03:33 ?        00:04:17 kube-controller-manager --allocate-node-cidrs=true --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf --authorization-kubeconfig=/etc
root         730     508  5 03:33 ?        00:08:10 kube-apiserver --advertise-address=172.18.0.2 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --ena
root         876       1  0 03:33 ?        00:00:02 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id b1296c9a7b2565f73869fb838b4139774631032b0c0565569f9216e0a6b3e80d -address /run/containerd
65535        897     876  0 03:33 ?        00:00:00 /pause
root         949       1  0 03:33 ?        00:00:02 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 3a956ee43a9f0c8b9e5cacc8ed737e9284350fbc1a1db676d71accf159d8223e -address /run/containerd
root         988       1  0 03:33 ?        00:00:02 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 36fa6e24be821741e714adf0d84878a824f99b310c67f1f4fc4c63da423c6faa -address /run/containerd
65535       1006     949  0 03:33 ?        00:00:00 /pause
65535       1015     988  0 03:33 ?        00:00:00 /pause
root        1087       1  0 03:33 ?        00:00:02 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id a02f950150207793f7cd5a646ddd3287bdec159c1b16b2f910adf6242a9497d0 -address /run/containerd
65535       1108    1087  0 03:33 ?        00:00:00 /pause
root        1129       1  0 03:33 ?        00:00:02 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id c8dda844968e8e6aa6de19d3eea94047e59a1022745afc1afa22de7af3eebafd -address /run/containerd
root        1201       1  0 03:33 ?        00:00:02 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id cb1a7f7f700a33b1a547d0de33b42d64f5c8423bb79d307e0b8ded734e15ad0c -address /run/containerd
65535       1219    1129  0 03:33 ?        00:00:00 /pause
65535       1227    1201  0 03:33 ?        00:00:00 /pause
root        1353     876  0 03:33 ?        00:00:19 /coredns -conf /etc/coredns/Corefile
root        1422     949  0 03:33 ?        00:00:02 /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf --hostname-override=kind-control-plane
root        1593    1129  0 03:33 ?        00:00:02 /bin/kindnetd
root        1601    1087  0 03:33 ?        00:00:00 /bin/sh -c /run.sh $FLUENTD_ARGS
root        1621    1601  0 03:33 ?        00:00:00 /bin/sh /run.sh
root        1625    1621  0 03:33 ?        00:00:00 /usr/bin/ruby2.3 /usr/local/bin/fluentd
root        1644    1625  0 03:33 ?        00:00:03 /usr/bin/ruby2.3 /usr/local/bin/fluentd
root        1721    1201  0 03:34 ?        00:00:18 /coredns -conf /etc/coredns/Corefile
root        2083     988  0 03:35 ?        00:00:07 local-path-provisioner --debug start --helper-image docker.io/kindest/local-path-helper:v20220512-507ff70b --config /etc/config/config.json
root        3694       1  0 05:43 ?        00:00:00 /usr/local/bin/containerd-shim-runc-v2 -namespace k8s.io -id 7121d69c49c86ae203034aacdadb07e88201861c712104ee90627e6ff1c99f93 -address /run/containerd
65535       3714    3694  0 05:43 ?        00:00:00 /pause
root        3868       0  0 05:52 pts/1    00:00:00 bash
root        3899    3868  0 05:54 pts/1    00:00:00 ps -ef
```

看来是没有 `dockerd` 的,那就是没法使用 `docker` 命令了,由于 `containerd` 是一个兼容 `CRI` 的容器运行时，我们可以尝试在容器中查找 `crictl`,任何 `CRI` 运行时的 `crictl` 都是 `dockerd` 守护进程的 `docker` 命令行工具

```bash
crictl images
docker.io/kindest/kindnetd                  v20220510-4929dd75   6fb66cd78abfe       45.2MB
docker.io/kindest/local-path-helper         v20220512-507ff70b   64623e9d887d3       2.86MB
docker.io/kindest/local-path-provisioner    v0.0.22-kind.0       4c1e997385b8f       17.4MB
docker.io/library/mysql                     5.7                  c20987f18b130       454MB
docker.io/netonline/fluentd-elasticsearch   v2.0.4               1d0aa76ccb08f       144MB
docker.io/netonline/fluentd-elasticsearch   v2.4.0               c90f770b63904       142MB
docker.io/resouer/ubuntu-bc                 latest               9f425d93eaee2       130MB
docker.io/yizhiyong/xtrabackup              latest               c415dbd7af07a       273MB
gcr.io/google-samples/xtrabackup            1.0                  c415dbd7af07a       273MB
k8s.gcr.io/coredns/coredns                  v1.8.6               a4ca41631cc7a       13.6MB
k8s.gcr.io/etcd                             3.5.3-0              aebe758cef4cd       102MB
k8s.gcr.io/kube-apiserver                   v1.24.0              9ef4b1de3be49       77.3MB
k8s.gcr.io/kube-controller-manager          v1.24.0              efa8a439d1460       65.6MB
k8s.gcr.io/kube-proxy                       v1.24.0              6960c0e47829d       112MB
k8s.gcr.io/kube-scheduler                   v1.24.0              41f5241e3396e       52.3MB
k8s.gcr.io/pause  
```

确实是有加载的镜像的,那么只有看看`k8s`那层镜像拉取机制是啥了,查查资料,发现是镜像标签的问题
我使用的是 `latest` 标签,那么镜像的拉取策略就是: `Always`

具体的可以查看[官方文档-镜像](https://kubernetes.io/zh-cn/docs/concepts/containers/images/)

那么解决方案就很简单

* 指定镜像 `tag`
* 指定镜像的拉取策略 `imagePullPolicy`

## 进行总结

其实 `Kind` 的[官方文档](https://kind.sigs.k8s.io/docs/user/quick-start/#loading-an-image-into-your-cluster)有介绍的,建议是以后遇到问题先仔细阅读官方文档

```text
NOTE: The Kubernetes default pull policy is IfNotPresent unless the image tag is :latest or omitted (and implicitly :latest) in which case the default policy is Always. IfNotPresent causes the Kubelet to skip pulling an image if it already exists. If you want those images loaded into node to work as expected, please:

don't use a :latest tag
and / or:

specify imagePullPolicy: IfNotPresent or imagePullPolicy: Never on your container(s).
See Kubernetes imagePullPolicy for more information.
```
