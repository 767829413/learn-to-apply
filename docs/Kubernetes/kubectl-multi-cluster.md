# kubectl命令行:多集群管理

![结构字段解析](https://s2.loli.net/2023/03/19/KvZOp6NqonyUe95.png)

```yml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
    server: https://192.168.148.61:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: 
    client-key-data: 
```

```bash
#设置集群参数
kubectl config set-cluster kubernetes \
 --server=https://ip:port \
 --certificate-authority=/etc/kubernetes/pki/ca.crt \
 --embed-certs=true \
 --kubeconfig=config

#设置上下文参数
kubectl config set-context test \
 --cluster=kubernetes \
 --user=cluster-admin \
 --kubeconfig=config

#设置默认上下文
kubectl config use-context test --kubeconfig=config

#设置客户端认证参数
kubectl config set-credentials cluster-admin \
 --certificate-authority=/etc/kubernetes/pki/ca.crt \
 --embed-certs=true \
 --client-key=/etc/kubernetes/pki/admin-key.pem \
 --client-certificate=/etc/kubernetes/pki/admin.pem \
 --kubeconfig=config

 #切换集群环境
 kubectl config --kubeconfig={config} use-context {name}
```

`--grace-period=0 --force 强制删除资源`