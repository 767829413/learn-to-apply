# kubeadm初始化k8s证书过期解决方案

`查看证书有效时间：`

```bash
openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -text  |grep Not
            Not Before: Apr 10 04:53:39 2023 GMT
            Not After : Apr  7 04:53:39 2033 GMT
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text |grep Not
            Not Before: Apr 10 04:53:39 2023 GMT
            Not After : Apr  9 04:53:39 2024 GMT
```

`延长证书过期时间：`

1. 把update-kubeadm-cert.sh文件上传到master1节点

2. 给update-kubeadm-cert.sh证书授权可执行权限

  ```bash
  chmod +x update-kubeadm-cert.sh
  ```

3. 执行下面命令，修改证书过期时间，把时间延长到10年

  ```bash
  ./update-kubeadm-cert.sh all
  ```

4. 在master1节点查询pod是否正常,能查询出数据说明证书签发完成

  ```bash
  kubectl get pods -n kube-system
  ```

5. 再次查看证书有效期，可以看到会延长到10年

  ```bash
  openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -text  |grep Not
  openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text  |grep Not
  ```