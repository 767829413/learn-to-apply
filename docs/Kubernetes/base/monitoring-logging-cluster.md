# kubernetes监控与日志管理

## 1. 查看集群资源状况

```bash
# 集群整体状态
kubectl cluster-info
# 更多集群信息
kubectl cluster-info dump
# 查看节点信息
kubectl get node
# 查看master组件状态
kubectl get cs
# 查看资源详情
kubectl describe <资源> <名称>
# 查看资源信息
kubectl get pod<Pod名称> -o wide -w
```

## 2. 监控集群资源利用率

`Metrics-server + cAdvisor 监控集群资源消耗`
![333050.png](https://s2.loli.net/2023/03/19/vrauWFZM39mYRI5.png)

[先安装组件metrics-server](https://github.com/kubernetes-sigs/metrics-server/tree/master)

```yml
containers:
- name: metrics-server
  image: bitnami/metrics-server:0.5.2 # 国内镜像替换
  imagePullPolicy: IfNotPresent
  args:
    - --cert-dir=/tmp
    - --secure-port=4443
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --kubelet-use-node-status-port
    - --metric-resolution=15s
    - --kubelet-insecure-tls ## 不使用https

```

```bash
# kubectl top [node | pod] -h 更多帮助
#查看Node资源消耗：
kubectl top node <node name>
#查看Pod资源消耗：
kubectl top pod <pod name>
```

## 3. 管理K8S组件,应用日志

```bash
# kubeadm安装的K8S除了 kubelet 都是容器化安装
journalctl -u kubelet
# 查看其他组件pod日志
kubectl get pod -n kube-system
kubectl logs xxxx -n kube-system

# 系统日志
cat /var/log/messages

# 查看容器标准输出日志
kubectl logs <Pod名称>
kubectl logs -f <Pod名称>
kubectl logs -f <Pod名称> -c <容器名称>
```

总结:

* 监控kubectl top -> apiserver -> metrics-server pod -> kubelet(cadvisor)

* 容器日志:
  * 输出到控制台(标准输出、标准错误): /var/lib/docker/containers/<container-id>/xxx-json.log
  * 落地到具体日志文件:通过emptydir挂载日志目录到宿主机 /ar/lib/kubelet/pods/<pod-id>/volumes/kubernetes,iorempty-dir/

* 日志
  * 容器化 (kubectl logs): 输出到控制台 (lgs)，日志文件(进入容器查看，)
  * systemd (journalctl -u xxx)