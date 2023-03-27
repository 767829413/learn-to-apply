# Kubernetes存储

## 1. 为什么需要存储卷？

容器部署过程中一般有以下三种数据：

* 启动时需要的初始数据，可以是配置文件
* 启动过程中产生的临时数据，该临时数据需要多个容器间共享
* 启动过程中产生的持久化数据

![存储卷](https://pic.imgdb.cn/item/6420f5e1a682492fccea0ccc.png)

## 2. 数据卷概述

[支持卷类型](https://kubernetes.io/zh-cn/docs/concepts/storage/volumes/#volume-types)

* Kubernetes中的Volume提供了在容器中挂载外部存储的能力
* Pod需要设置卷来源（spec.volume）和挂载点（spec.containers.volumeMounts）两个信息后才可
以使用相应的Volume
* 本地: hostPath, emptyDir
* 网络: nfs, cephfs, rbd
* k8s本身存储: configMap, downwardAPI, secret

## 3. 临时存储卷，节点存储卷，网络存储卷

1. 临时存储卷

创建一个空卷，挂载到Pod中的容器。Pod删除该卷也会被删除。

应用场景：Pod中容器之间数据共享

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: write
    image: centos
    command: ["bash","-c","for i in {1..100};do echo $i >> /data/hello;sleep 1;done"]
    volumeMounts:
    - name: data
      mountPath: /data
  - name: read
    image: centos
    command: ["bash","-c","tail -f /data/hello"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
```

```bash
[root@master ~]# kubectl get po -o wide
NAME                   READY   STATUS    RESTARTS   AGE     IP               NODE    NOMINATED NODE   READINESS GATES
mypod                  2/2     Running   1          3m12s   10.244.104.38    node2   <none>           <none>

[root@node2 ~]# docker ps | grep mypod
901d7371c36d        centos                                              "bash -c 'for i in {…"   59 seconds ago      Up 58 seconds                           k8s_write_mypod_default_7cf97541-273e-49c9-89e2-453079d73f8d_1
b6c840bff00d        centos                                              "bash -c 'tail -f /d…"   2 minutes ago       Up 2 minutes                            k8s_read_mypod_default_7cf97541-273e-49c9-89e2-453079d73f8d_0
d1679b693bf7        registry.aliyuncs.com/google_containers/pause:3.2   "/pause"                 3 minutes ago       Up 3 minutes                            k8s_POD_mypod_default_7cf97541-273e-49c9-89e2-453079d73f8d_0

[root@node2 ~]# cd /var/lib/kubelet/pods/7cf97541-273e-49c9-89e2-453079d73f8d/volumes/kubernetes.io~empty-dir/data/

[root@node2 data]# tail -n 1 hello 
100
```

2. 节点存储卷

挂载Node文件系统上文件或者目录到Pod中的容器。

应用场景：Pod中容器需要访问宿主机文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod2
spec:
  containers:
  - name: busybox
    image: busybox
    args:
    - /bin/sh
    - -c
    - sleep 36000
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: /tmp
      type: Directory
```

```bash
[root@master ~]# kubectl get po -o wide
NAME                   READY   STATUS    RESTARTS   AGE    IP               NODE    NOMINATED NODE   READINESS GATES
mypod2                 1/1     Running   0          11s    10.244.104.39    node2   <none>           <none>
[root@master ~]# kubectl exec -it mypod2 -- sh
/ # cd data/
/data # echo 123455 > 123.log

[root@node2 data]# cat /tmp/123.log 
123455
```

3. 网络存储卷

首先找一台虚拟机配置NFS

```bash
[root@extra ~]# yum install nfs-utils -y
# 共享的目录创建
[root@extra ~]# mkdir -p /ifs/k8s
# /ifs/k8s *(rw,no_root_squash) *:表示任意ip访问 (rw,root_no_squash)权限和访问账号类型
[root@extra ~]# vim /etc/exports

# 启动nfs服务
[root@extra ~]# systemctl start nfs
# 查看服务是否启动成功
[root@extra ~]# ps -ef | grep nfs
root      16608      2  0 05:49 ?        00:00:00 [nfsd4_callbacks]
root      16614      2  0 05:49 ?        00:00:00 [nfsd]
root      16615      2  0 05:49 ?        00:00:00 [nfsd]
root      16616      2  0 05:49 ?        00:00:00 [nfsd]
root      16617      2  0 05:49 ?        00:00:00 [nfsd]
root      16618      2  0 05:49 ?        00:00:00 [nfsd]
root      16619      2  0 05:49 ?        00:00:00 [nfsd]
root      16620      2  0 05:49 ?        00:00:00 [nfsd]
root      16621      2  0 05:49 ?        00:00:00 [nfsd]
root      16632  16282  0 05:49 pts/0    00:00:00 grep --color=auto nfs

# 挂载测试一下,要先安装 nfs-utils
[root@master ~]# mount -t nfs extra:/ifs/k8s /mnt
[root@master ~]# cd /mnt && touch 123.txt
# 查看文件是否存在
[root@extra ~]# ls /ifs/k8s
123.txt
# 卸载已挂载
umount /mnt/
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        project: blog
    spec:
      volumes: 
      - name: wwwroot
        nfs: 
          server: extra
          path: /ifs/k8s
      containers:
      - image: nginx
        name: nginx
        ports: 
        - containerPort: 80
        volumeMounts: 
        - name: wwwroot
          mountPath: /usr/share/nginx/html
```

## 4. 持久卷概述

## 5. PV 静态供给

## 6. PV 动态供给

## 7. 案例：应用程序使用持久卷存储数据

## 8. 有状态应用部署：StatefulSet 控制器

## 9. 应用程序配置文件存储：ConfigMap

## 10. 敏感数据存储：Secret