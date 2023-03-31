# Kubernetes集群维护

## 1. Bootstrap Token 方式增加 Node

    TLS Bootstraping：在kubernetes集群中，Node上组件kubelet和kube-proxy都需要与kube-apiserver进行通信，为了增加传输安全性，采用https方式。这就涉及到Node组件需要具备kube-apiserver用的证书颁发机构（CA）签发客户端证书，当规模较大时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了简化流程，Kubernetes引入了TLS bootstraping机制来自动颁发客户端证书，所以强烈建议在Node上使用这种方式。

## 2. K8s 集群证书续签（kubeadm）

## 3. K8s 数据库 Etcd 备份与恢复