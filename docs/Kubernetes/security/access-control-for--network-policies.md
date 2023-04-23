# 网络策略的访问控制

## 网络策略概述

```text
默认情况下，Kubernetes 集群网络没任何网络限制，Pod 可以与任何其他 Pod 通信，在某些场景下就需要进行网络控制，减少网络攻击面，提高安全性，这就会用到网络策略.网络策略 (Network Policy) : 是一个K8s资源，用于限制Pod出入流量，提供Pod级别和Namespace级别网络访问控制
```

NetworkPolicy 的示例: 

```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default # 网络策略应用到的命名空间
spec:
  podSelector: # 网络策略的目标pod
    matchLabels:
      role: db
  policyTypes: # 策略类型,指定策略用于出站还是进站流量
  - Ingress
  - Egress
  ingress: # 外部可以访问pod的限制
  - from:
    - ipBlock: # ip段
        cidr: 172.17.0.0/16
        except:
          - 172.17.1.0/24
    - namespaceSelector: # 命名空间
        matchLabels:
          project: myproject # 命名空间也有标签,这里是基于命名空间的标签选定
    - podSelector: # 目标pod,根据标签选择
        matchLabels: 
          role: frontend # pod的标签
    ports:
    - protocol: TCP
      port: 6379
  egress: # pod可以访问外部限制,与上面设置类似
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
      ports:
        - protocol: TCP
          port: 5978
```

## 网络策略工作流程

1. 创建Network Policy资源
2. Policy Controller监控网络策略，同步并通知节点上程序
3. 节点上DaemonSet运行的程序从etcd中获取Policy，调用本地Iptables创建防火墙规则

![示意图](https://pic1.imgdb.cn/item/6441f2a60d2dde5777b52a0d.png)

## 网络策略应用场景

- 应用程序间的访问控制，例如项目A不能访问项目B的Pod
- 开发环境命名空间不能访问测试环境命名空间Pod
- 当Pod暴露到外部时，需要做Pod白名单
- 多租户网络环境隔离

## 网络访问控制5个案例

`案例1：拒绝命名空间下所有Pod出入站流量`

需求：拒绝 test 命名空间下所有Pod入、出站流量

```bash
cat > ./deny-all.yml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: test 
spec:
  podSelector: {} # 选择所有pod
  policyTypes: 
  - Ingress
  - Egress
# ingress egress 都没有指定规则,不允许任何流量进出pod
EOF
kubectl create namespace test
kubectl run busybox -n test --image=busybox:1.28 --restart=Never -- sleep 1h
kubectl get pod -n test -o wide
NAME      READY   STATUS    RESTARTS   AGE     IP              NODE     NOMINATED NODE   READINESS GATES
busybox   1/1     Running   0          2m24s   10.244.84.143   node-1   <none>           <none>
# 策略未生效前

# 测试访问外部
kubectl exec busybox -n test -- ping baidu.com
PING baidu.com (110.242.68.66): 56 data bytes
64 bytes from 110.242.68.66: seq=0 ttl=127 time=32.410 ms
...
# 测试外部pod访问
kubectl run busybox1 --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # ping 10.244.84.143
PING 10.244.84.143 (10.244.84.143): 56 data bytes
64 bytes from 10.244.84.143: seq=0 ttl=63 time=0.081 ms
...
# 测试内部pod访问
kubectl run busybox1 -n test --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # ping 10.244.84.143
PING 10.244.84.143 (10.244.84.143): 56 data bytes
64 bytes from 10.244.84.143: seq=0 ttl=63 time=0.137 ms
...

# 策略生效后
kubectl apply -f deny-all.yml

# 测试访问外部
kubectl exec busybox -n test -- ping baidu.com
# 超时退出
# 测试外部pod访问
kubectl run busybox1 --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # ping 10.244.84.143
PING 10.244.84.143 (10.244.84.143): 56 data bytes
^C
--- 10.244.84.143 ping statistics ---
5 packets transmitted, 0 packets received, 100% packet loss
# 测试内部pod访问
kubectl run busybox1 -n test --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # ping 10.244.84.143
PING 10.244.84.143 (10.244.84.143): 56 data bytes
^C
--- 10.244.84.143 ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss
```

`案例2：拒绝其他命名空间Pod访问`

需求：test命名空间下所有pod可以互相访问，也可以访问其他命名空间Pod，但其他命名空间不能访问test命名空间Pod。

```bash
cat > ./deny-all-ns.yml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ns
  namespace: test
spec:
  podSelector: {}
  policyTypes: 
  - Ingress
  - Egress
  ingress: 
  - from:
    - podSelector: {} # 匹配本命名空间所有pod
  egress:
  - {}
EOF
kubectl run busybox -n test --image=busybox:1.28 --restart=Never -- sleep 1h
kubectl get pod -n test -o wide
NAME      READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
busybox   1/1     Running   0          6s    10.244.84.149   node-1   <none>           <none>
# 策略未生效前

# 测试访问外部
kubectl exec busybox -n test -- ping baidu.com
PING baidu.com (110.242.68.66): 56 data bytes
64 bytes from 110.242.68.66: seq=0 ttl=127 time=32.410 ms
...
# 测试外部pod访问
kubectl run busybox1 --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # ping 10.244.84.149
PING 10.244.84.149 (10.244.84.149): 56 data bytes
64 bytes from 10.244.84.149: seq=0 ttl=63 time=0.099 ms
...
# 测试内部pod访问
kubectl run busybox1 -n test --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # ping 10.244.84.149
PING 10.244.84.149 (10.244.84.149): 56 data bytes
64 bytes from 10.244.84.149: seq=0 ttl=63 time=0.105 ms
...

# 策略生效后
kubectl apply -f deny-all-ns.yml

# 测试访问外部
kubectl exec busybox -n test -- ping baidu.com
PING baidu.com (39.156.66.10): 56 data bytes
64 bytes from 39.156.66.10: seq=0 ttl=127 time=27.302 ms
...
# 测试外部pod访问
kubectl run busybox1 --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # ping 10.244.84.149
PING 10.244.84.149 (10.244.84.149): 56 data bytes
^C
--- 10.244.84.149 ping statistics ---
6 packets transmitted, 0 packets received, 100% packet loss
# 测试内部pod访问
kubectl run busybox1 -n test --image busybox:1.28 --restart=Never --rm -it busybox -- sh
/ # ping 10.244.84.149
PING 10.244.84.149 (10.244.84.149): 56 data bytes
64 bytes from 10.244.84.149: seq=0 ttl=63 time=0.194 ms
```

`案例3：允许其他命名空间Pod访问指定应用`

需求：允许其他命名空间访问test命名空间指定Pod 标签 role=db

```bash
cat > ./target-pod-allow-all-ns.yml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: target-pod-allow-all-ns
  namespace: test # 网络策略应用到的命名空间
spec:
  podSelector: # 网络策略的目标pod
    matchLabels:
      role: db
  policyTypes: 
  - Ingress
  ingress: 
  - from:
    - namespaceSelector: {} # 匹配所有命名空间
EOF
```

`案例4：同一个命名空间下应用之间限制访问`

需求：将test命名空间中标签为run=web的pod隔离，只允许标签为run=client1的pod访问80端口

```bash
cat > ./target-pod-allow-target-pod.yml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: target-pod-allow-target-pod
  namespace: test # 网络策略应用到的命名空间
spec:
  podSelector: # 网络策略的目标pod
    matchLabels:
      run: web
  policyTypes: 
  - Ingress
  ingress: 
  - from:
    - podSelector:
        matchLabels:
          run: client1
    ports:
    - protocol: TCP
        port: 80
EOF
```

`案例5：只允许指定命名空间中的应用访问`

需求：限制dev命名空间标签为env=dev的pod，只允许prod命名空间中的pod访问和其他所有命名空间app=client1标签pod访问

```bash
kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   43h   kubernetes.io/metadata.name=default
dev               Active   43h   kubernetes.io/metadata.name=dev
prod              Active   43h   kubernetes.io/metadata.name=prod
cat > ./target-ns-pod-allow-target-ns-pod.yml <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: target-ns-pod-allow-target-ns-pod
  namespace: dev # 网络策略应用到的命名空间
spec:
  podSelector: # 网络策略的目标pod
    matchLabels:
      env: dev
  policyTypes: 
  - Ingress
  ingress: 
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name=prod
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app: client1
EOF
```