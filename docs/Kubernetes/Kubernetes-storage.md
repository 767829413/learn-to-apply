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

* PersistentVolume（PV）：对存储资源创建和使用的抽象，使得存储作为集群中的资源管理
  * 静态
  * 动态
* PersistentVolumeClaim（PVC）：让用户不需要关心具体的Volume实现细节

![pv<-pvc<-pod](https://pic.imgdb.cn/item/64224800a682492fcce44eb4.png)

PV与PVC:

1. 一对一的关系
2. PV可以是多个后端存储
3. PV一般都是k8s运维创建(存储池)

访问模式:
<https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#access-modes>

* ReadWriteOnce(RWO)
  卷可以被一个节点以读写方式挂载。 ReadWriteOnce 访问模式也允许运行在同一节点上的多个 Pod 访问卷。
* ReadOnlyMany(ROX)
  卷可以被多个节点以只读方式挂载。
* ReadWriteMany(RWX)
  卷可以被多个节点以读写方式挂载。
* ReadWriteOncePod(RWOP)
  卷可以被单个 Pod 以读写方式挂载。 如果你想确保整个集群中只有一个 Pod 可以读取或写入该 PVC， 请使用 ReadWriteOncePod 访问模式。这只支持 CSI 卷以及需要 Kubernetes 1.22 以上版本。

PV 和 PVC 匹配:

1. 访问模式
2. 存储容量

## 5. PV 静态供给

![介绍](https://pic.imgdb.cn/item/642248e8a682492fcce57e64.png)

Kubernetes支持持久卷的存储插件：
<https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: pd
  name: pd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pd
  template:
    metadata:
      labels:
        app: pd
    spec:
      volumes: 
      - name: wwwroot
        persistentVolumeClaim:
          claimName: my-pvc
      containers:
      - image: nginx
        name: nginx
        ports: 
        - containerPort: 80
        volumeMounts: 
        - name: wwwroot
          mountPath: /usr/share/nginx/html

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /ifs/k8s/pv1
    server: extra

--- 

apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv2
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /ifs/k8s/pv2
    server: extra

---

apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv3
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: /ifs/k8s/pv3
    server: extra
```

## 6. PV 动态供给

![结构](https://pic.imgdb.cn/item/642261d5a682492fcc0be6e1.png)

Dynamic Provisioning机制工作的核心在于StorageClass的API对象。

StorageClass声明存储插件，用于自动创建PV。

Kubernetes支持动态供给的存储插件：

<https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/>

![动态供给-NFS](https://pic.imgdb.cn/item/6422819aa682492fcc3eceac.png)

<https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner>

## 7. 案例：应用程序使用持久卷存储数据

<https://github.com/kubernetes-retired/external-storage/tree/master/nfs-client/deploy>
使用文件 (class.yaml, deployment.yaml, rbac.yaml)

```bash
[root@master ~]# kubectl get sc
NAME                  PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   fuseim.pri/ifs   Delete          Immediate           false                  24m
```

PersistentVolumeClaim 类型文件

添加 storageClassName: managed-nfs-storage

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: pd
  name: pd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pd
  template:
    metadata:
      labels:
        app: pd
    spec:
      volumes: 
      - name: wwwroot
        persistentVolumeClaim:
          claimName: my-pvc
      containers:
      - image: nginx
        name: nginx
        ports: 
        - containerPort: 80
        volumeMounts: 
        - name: wwwroot
          mountPath: /usr/share/nginx/html


---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: managed-nfs-storage
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi

```

## 8. 有状态应用部署：StatefulSet 控制器

有状态和无状态应用区别: 

有状态: 数据存储,固定网络ID(例如分布式应用: etcd, zk, mysql), pod都不对等,客户端针对特定pod工作
无状态: 无数据,无需固定网络ID(例如web, 网站, api),所有pod都是对等的,通过LB负载均衡器提供工作

StatefulSet：

* 部署有状态应用
* 解决Pod独立生命周期，保持Pod启动顺序和唯一性
  1. 稳定，唯一的网络标识符，持久存储
  2. 有序，优雅的部署和扩展、删除和终止
  3. 有序，滚动更新

应用场景：数据库

1. StatefulSet：稳定的网络ID

    多了一个serviceName: “nginx”字段，这就告诉StatefulSet控制器要使用nginx这个headless service来保证Pod的身份

    ```yaml
    # 普通service
    apiVersion: v1
    kind: Service
    metadata: 
      name: my-svc
    spec: 
      selector: 
        app: nginx
      ports: 
      - protocol: TCP
        port: 80
        targetPort: 9376
    ---
    # Headless services
    apiVersion: v1
    kind: Service
    metadata: 
      name: my-headless 
    spec: 
      clusterIP: None
      selector: 
        app: nginx
      ports: 
      - protocol: TCP
        port: 80
        targetPort: 9376
    
    ---

    apiVersion: apps/v1
    kind: StatefulSet 
    metadata:
      labels:
        app: web
      name: web
    spec:
      replicas: 3
      serviceName: my-headless 
      selector:
        matchLabels:
          app: web
      template:
        metadata:
          labels:
            app: web
        spec:
          containers:
          - image: nginx
            name: nginx
    ```

    ClusterIP A记录格式：

    service-name.namespace-name.svc.cluster.local

    ClusterIP=None A记录格式：

    statefulsetName-index.service-name.namespace-name.svc.cluster.local

    示例：web-0.nginx.default.svc.cluster.local 

2. StatefulSet：稳定的存储

    StatefulSet的存储卷使用VolumeClaimTemplate创建，称为卷申请模板，当StatefulSet使用VolumeClaimTemplate创建一个[PersistentVolume](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/)时，同样也会为每个Pod分配并创建一个编号的PVC。

    ```yaml
    apiVersion: apps/v1
    kind: StatefulSet 
    metadata:
      labels:
        app: web
      name: web
    spec:
      replicas: 3
      serviceName: my-headless 
      selector:
        matchLabels:
          app: web
      template:
        metadata:
          labels:
            app: web
        spec:
          containers:
          - image: nginx
            name: nginx
            volumeMounts: 
            - name: www
              mountPath: /usr/share/nginx/html
      volumeClaimTemplates:
      - metadata:
          name: www
        spec:
          accessModes: [ "ReadWriteOnce" ]
          storageClassName: "managed-nfs-storage"
          resources:
            requests:
              storage: 1Gi
    ```

3. StatefulSet：小结

    StatefulSet与Deployment区别：有身份的！

    身份三要素：
    * 域名
    * 主机名
    * 存储（PVC）

## 9. 应用程序配置文件存储：ConfigMap

pod中变量的定义:

1. 自定义变量
2. 从 secret, configmap获取
3. pod属性

[通过环境变量将 Pod 信息呈现给容器](https://kubernetes.io/zh-cn/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)

数据存储在Etcd中，让Pod中容器以Volume或者变量方式访问。

应用场景：应用程序配置

Pod使用configmap两种方式：

* 变量注入: 主要适用少的数据,以键值存储

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myconf
    data:
      special.level: info
      special.type: hello
    
    --- 

    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod1
    spec:
      restartPolicy: Never
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh", "-c", "echo $(LEVEL) $(TYPE) $(POD_NAME) $(MY_SET)"]
          env:
          - name: LEVEL
            valueFrom:
              configMapKeyRef:
                name: myconf
                key: special.level
          - name: TYPE
            valueFrom:
              configMapKeyRef:
                name: myconf
                key: special.type
          - name: POD_NAME
            valueFrom: 
              fieldRef: 
                fieldPath: metadata.name
          - name: MY_SET
            value: 123HI
    ```

    ```bash
    [root@master configmap]# kubectl logs mypod1
    info hello mypod1 123HI
    ```

* 数据卷挂载: 适用于配置文件

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myconf2
    data:
      refis.properties: |
        redis.host=127.0.0.1
        redis.port=6379
        redis.password=123456
    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod2
    spec:
      restartPolicy: Never
      volumes: 
      - name: config
        configMap: 
          name: myconf2
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh", "-c", "cat /etc/config/refis.properties"]
          volumeMounts:
          - name: config
            mountPath: "/etc/config"
    ```

    ```bash
    [root@master configmap]# kubectl logs mypod2
     redis.host=127.0.0.1
     redis.port=6379
     redis.password=123456
    ```

## 10. 敏感数据存储：Secret

<https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/>

与ConfigMap类似，区别在于Secret主要存储敏感数据，所有的数据要经过编码。

应用场景：凭据

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: eGlhb21pbmcK
  password: MTIzNDU2Cg==

---

apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: nginx
    image: nginx
    env: 
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: username
    - name: SECRET_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: password

---

apiVersion: v1
kind: Pod
metadata:
  name: mypod2
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```