# minio调研分享

## minio简介

MinIO 是一个高性能的对象存储系统，兼容 Amazon S3 API，适用于私有云和混合云环境。其特点包括简单易用、支持大规模集群部署、高性能和高可用性。

相关文档: <https://min.io/docs/minio/kubernetes/upstream/index.html>

## minio单节点k8s部署

基于实践来说，必须要使用一番，为了快速验证和当前私有化业务的匹配度，同时自己也有测试用的`k8s`集群，那就整个单节点体验一下，参考文档是基于这个官方的`docker`部署文档: <https://min.io/docs/minio/container/operations/install-deploy-manage/deploy-minio-single-node-single-drive.html>

剩下的就是朴实无华的`yaml`编写:

1. 首先要准备`pv`和`pvc`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  namespace: minio-dev
  name: minio-local-pv
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  local:
    path: /mnt/disks/minio
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: minio-dev
  name: minio-local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 20Gi
```

2. 剩下就是对 `namespace`, `pod` 和 `service`的编写

```yaml
# Deploys a new Namespace for the MinIO Pod
apiVersion: v1
kind: Namespace
metadata:
  name: minio-dev # Change this value if you want a different namespace name
  labels:
    name: minio-dev # Change this value to match metadata.name
---
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: minio
  name: minio
  namespace: minio-dev # Change this value to match the namespace metadata.name
spec:
  containers:
  - name: minio
    image: quay.io/minio/minio:latest
    command:
    - /bin/bash
    - -c
    args: 
    - minio server /data --console-address :9001
    ports:
    - containerPort: 9000
      protocol: TCP
    - containerPort: 9001
      protocol: TCP
    env:
    - name: MINIO_ROOT_USER
      value: "root"
    - name: MINIO_ROOT_PASSWORD
      value: "Fy04030201"
    volumeMounts:
    - mountPath: /data
      name: localvolume # 对应于 'spec.volumes' 持久卷
  nodeSelector:
    kubernetes.io/hostname: node1 # Specify a node label associated to the Worker Node on which you want to deploy the pod.
  volumes:
  - name: localvolume
    persistentVolumeClaim:
      claimName: minio-local-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio-service
  namespace: minio-dev
spec:
  type: NodePort
  selector:
    app: minio
  ports:
    - name: http
      port: 9000
      targetPort: 9000
      nodePort: 30001
      protocol: TCP
    - name: console
      port: 9001
      targetPort: 9001
      nodePort: 30002
      protocol: TCP
```

3. 把对应`yaml`放到一个`minio`文件夹

```bash
cd ./minio
kubectl apply -f ./*
```

![3.png](https://s2.loli.net/2025/01/08/PSV9QhgqeDb3urz.png)
![2.png](https://s2.loli.net/2025/01/08/RMGDXkiwCbamJ74.png)
![5.png](https://s2.loli.net/2025/01/08/YgiPmAnUpZyTv9z.png)
![4.png](https://s2.loli.net/2025/01/08/7z2QxipsXNPCKHd.png)

4. 等待部署完毕

![1.png](https://s2.loli.net/2025/01/08/LxWEmAPeOtkbcJY.png)
![8.png](https://s2.loli.net/2025/01/08/m1X46tWVefzHMsK.png)
![7.png](https://s2.loli.net/2025/01/08/WYoikfRzJGPBEra.png)
![9.png](https://s2.loli.net/2025/01/08/bASCKwRePUGgv2N.png)

5. 登录管理台

管理台地址: <http://your_ip_or_domain:30002>

![6.png](https://s2.loli.net/2025/01/08/6C5BoMrI89S2pVw.png)

## 高可用

生产环境中部署 `MinIO` 对象存储解决方案可以采用`多节点多驱动（MNMD）`配置，以确保高性能、高可用性和高可扩展性。下面的文档介绍了 `MinIO` 的产品特性，如擦除编码、版本控制、加密、可观测性等。

同时也包含说明了部署前的网络和防火墙配置要求，强调了节点间的全双工网络访问、负载均衡器的使用以及推荐的负载均衡器类型（如 `NGINX`、`HAProxy`）。此外，还有配置顺序式的主机名和 `IP` 地址，以及存储要求，包括使用本地存储、`XFS` 格式化驱动器、一致的驱动器类型和大小，以及确保独占的驱动器访问权限。

内存方面，建议至少配置 `32GiB` 内存，并且需要保证时间同步。这里也有擦除编码的冗余性设置，建议使用 `MinIO` 提供的擦除编码计算器来规划部署。

需要按照具体的步骤来安装 `MinIO` 二进制文件、配置 `systemd` 服务文件、添加 `TLS/SSL` 证书以及启动 `MinIO` 服务，并通过浏览器访问 `MinIO Console` 进行管理。

参考文档:

<https://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-multi-node-multi-drive.html>

使用内置的`错误编码（Erasure Coding）`来提供数据冗余、容错性和高可用性。

`MinIO` 部署可以容忍多个驱动器和节点的故障，最多可以失去一半的驱动器和节点而不会丢失数据。

为了更好地规划部署，`MinIO` 提供了在线错误编码计算器。

下面参考资料主要介绍了如何管理和维护正在失败的驱动器，包括如何尝试将其恢复到良好状态，或者如何替换它们而不影响现有操作。

`MinIO` 允许在不中断服务的情况下热插拔驱动器，并且不需要昂贵的硬件和 `RAID` 控制器。

在驱动器损坏无法修复的情况下，可以用新驱动器替换损坏的驱动器，`MinIO` 会使用数据或校验片段来修复损坏的片段。修复过程对用户应用程序透明，不会影响整个集群的性能。`MinIO` 还改进了修复速度，例如避免在磁盘修复过程中修复新上传的对象，以及同时修复多个磁盘。

参考文档:
<https://blog.min.io/troubleshooting-disk-failures/>

## GO-SDK

参考资料: <https://github.com/minio/minio-go/blob/master/docs/API.md>

这里我的主要实现可以在这查看:

<https://github.com/767829413/advanced-go/blob/main/storage/minio.go>

## minio鉴权

1. 不支持`object`设置权限

考资料: <https://github.com/minio/minio/issues/8195>

2. 访问密钥和密钥对

![10.png](https://s2.loli.net/2025/01/08/cMHVvu7FTpRCyro.png)

3. 安全令牌服务

参考资料: <https://github.com/minio/minio/blob/master/docs/sts/README.md>

4. AssumeRole

返回一组可用于访问 `MinIO` 资源的临时安全凭证。`AssumeRole` 需要 `mino` 上现有用户的授权凭证。上述使用的 `go sdk` 就是采用这种方式。

参考资料: <https://github.com/minio/minio/blob/master/docs/sts/assume-role.md>