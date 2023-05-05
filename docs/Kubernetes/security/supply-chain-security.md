# 供应链安全

## 可信任软件供应链

`概述`

```text
可信任软件供应链：指在建设基础架构过程中，涉及的软件都是可信任的。

在k8s领域可信软件供应链主要是指镜像，因为一些软件交付物都是镜像，部署的最小载体
```

![supply-images.png](https://s2.loli.net/2023/05/04/Q9H8TwWfFr5v1Ng.png)

## 构建镜像Dockerfile文件优化

* 减少镜像层：一次RUN指令形成新的一层，尽量Shell命令都写在一行，减少镜像层。
* 清理无用文件：清理对应的残留数据，例如yum缓存。
* 清理无用的软件包：基础镜像默认会带一些debug工具，可以删除掉，仅保留应用程序所需软件，防止黑客利用。
* 选择最小的基础镜像：例如alpine
* 使用非root用户运行：USER指令指定普通用户

```Dockerfile
FROM python
RUN useradd python
RUN mkdir /data/www -p
COPY . /data/www
RUN chown -R python /data
RUN pip install flask -i https://mirrors.aliyun.com/pypi/simple/
WORKDIR /data/www
USER python
CMD python main.py 
```

## 镜像漏洞扫描工具：Trivy

`介绍`

```text
Trivy：是一种用于容器镜像、文件系统、Git仓库的漏洞扫描工具。发现目标软件存在的漏洞。

Trivy易于使用，只需安装二进制文件即可进行扫描，方便集成CI系统。
```

*项目地址*：<https://github.com/aquasecurity/trivy>

`示例`

```bash
# 跳过更新
trivy image -h | grep skip
      --skip-dirs strings       specify the directories where the traversal is skipped
      --skip-files strings      specify the file paths to skip traversal
      --skip-db-update              skip updating vulnerability database
      --skip-java-db-update         skip updating Java index database
      --skip-policy-update          skip fetching rego policy updates
# 容器镜像扫描
trivy image nginx
trivy image -i nginx.tar
# 打印指定（高危、严重）漏洞信息
trivy image -s HIGH nginx
trivy image -s HIGH, CRITICAL nginx
# JSON格式输出并保存到文件
trivy image nginx -f json -o /root/output.json
```

## 检查YAML文件安全配置：kubesec

`介绍`

```text
kubesec：是一个针对K8s资源清单文件进行安全配置评估的工具，根据安全配置
最佳实践来验证并给出建议。
```

*官网：*<https://kubesec.io>

*项目地址：*<https://github.com/controlplaneio/kubesec>

`使用说明`

```bash
# 如果使用出现
[
  {
    "object": "Deployment/web.default",
    "valid": false,
    "fileName": "test-deploy.yaml",
    "message": "failed downloading schema at https://raw.githubusercontent.com/yannh/kubernetes-json-schema/master/master-standalone-strict/deployment-apps-v1.json: Get \"https://raw.githubusercontent.com/yannh/kubernetes-json-schema/master/master-standalone-strict/deployment-apps-v1.json\": dial tcp 0.0.0.0:443: connect: connection refused",
    "score": 0,
    "scoring": {}
  }
]
# 可以下载schema文件到本地,使用 --schema-location ~/kubesec/deployment-apps-v1.json 来指定
kubesec scan --schema-location ~/kubesec/deployment-apps-v1.json  test-deploy.yaml
# 扫描部署的yaml文件
kubesec scan deployment.yaml
# 使用容器环境执行检查
docker run -i kubesec/kubesec scan /dev/stdin < deployment.yaml

# kubesec内置一个HTTP服务器，可以直接启用，远程调用。
# 二进制
kubesec http 8080 &
# Docker容器
docker run -d -p 8080:8080 kubesec/kubesec http 8080
# 举例
curl -sSX POST --data-binary @deployment.yaml http://192.168.31.71:8080/scan
```

## 准入控制器： Admission Webhook

`介绍`

```text
Admission Webhook：准入控制器Webhook是准入控制插件的一种，用于拦截所有向APISERVER发送的请求，并且可以修改请求或拒绝请求。

Admission webhook为开发者提供了非常灵活的插件模式，在kubernetes资源持久化之前，管理员通过程序可以对指定资源做校验、修改等操作。例如为资源自动打标签、pod设置默认SA，自动注入sidecar容器等。
```

**相关Webhook准入控制器：**

* MutatingAdmissionWebhook：修改资源，理论上可以监听并修改任何经过ApiServer处理的请求
* ValidatingAdmissionWebhook：验证资源
* ImagePolicyWebhook：镜像策略，主要验证镜像字段是否满足条件

![web-hook.png](https://s2.loli.net/2023/05/04/tPNTdEW3orabOVp.png)

## 准入控制器： ImagePolicyWebhook

`介绍`

imagepolicywebhook: <https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook>

![imagePolicyWebHook.png](https://s2.loli.net/2023/05/04/KvXdAqbSBlYxPtm.png)

`使用`

*webhook服务器代码借鉴这个:* <https://github.com/kainlite/kube-image-bouncer>

```bash
# 添加准入控制器配置目录 /etc/kubernetes/image-policy/
sudo mkdir -p /etc/kubernetes/image-policy/
cd /etc/kubernetes/image-policy

# 生成自签证书
mkdir /self-signed/ && cd /self-signed
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

cat > webhook-csr.json <<EOF
{
  "CN": "webhook",
  "hosts": [
   "192.168.148.119" 
   ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes webhook-csr.json | cfssljson -bare webhook

cat > apiserver-client-csr.json <<EOF
{
  "CN": "apiserver",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes apiserver-client-csr.json | cfssljson -bare apiserver-client

# 将证书远程复制到 webhook 服务器 192.168.148.119
scp ./{webhook-key.pem,webhook.pem} root@192.168.148.119:/root/

# 在 webhook 服务器 192.168.148.119 启动一个服务,这里是二进制 local
./local 
...
⇨ https server started on [::]:8080


# 策略服务器配置文件
sudo cat > connecct_webhook.yaml <<EOF
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/image-policy/self-signed/webhook.pem
    server: https://192.168.148.119:8080/image_policy
  name: webhook
contexts:
- context:
    cluster: webhook
    user: apiserver
  name: webhook
current-context: webhook
kind: Config
preferences: {}
users:
- name: apiserver
  user:
    client-certificate: /etc/kubernetes/image-policy/self-signed/apiserver-client.pem
    client-key: /etc/kubernetes/image-policy/self-signed/apiserver-client-key.pem
EOF

# 准入控制器配置文件
sudo cat > admission_configuration.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
  - name: ImagePolicyWebhook
    configuration:
      imagePolicy:
        kubeConfigFile: /etc/kubernetes/image-policy/connecct_webhook.yaml # 连接镜像策略服务器配置文件
        allowTTL: 50 # 控制批准请求的缓存时间，单位秒
        denyTTL: 50 # 控制批准请求的缓存时间，单位秒
        retryBackoff: 500 # 控制重试间隔，单位毫秒
        defaultAllow: true # 确定webhook后端失效时的行为
EOF

# 启用准入控制器(kubeadm引导集群)
# --enable-admission-plugins=NamespaceLifecycle,ImagePolicyWebhook
# --admission-control-config-file=/etc/kubernetes/image-policy/admission_configuration.yaml
# 打开主节点的 /etc/kubernetes/manifests/kube-apiserver.yaml 
sudo vim /etc/kubernetes/manifests/kube-apiserver.yaml
...
spec:
  containers:
  - command:
    - kube-apiserver
...
    - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
    - --admission-control-config-file=/etc/kubernetes/image-policy/admission_configuration.yaml
...
    volumeMounts:
    - mountPath: /etc/kubernetes/image-policy
      name: image-policy
      readOnly: true
    - mountPath: /etc/ssl/certs
      name: ca-certs
      readOnly: true
...
  volumes:
  - hostPath:
      path: /etc/kubernetes/image-policy
      type: DirectoryOrCreate
    name: image-policy
  - hostPath:
      path: /etc/ssl/certs
      type: DirectoryOrCreate
    name: ca-certs
...

# 可以通过docker 或 crictl 来查看宿主机的 apiserver 容器状态
sudo crictl ps  -a | grep kube-apiserver
b9dcc9fa46aa2       6f707f569b572       17 minutes ago      Running             kube-apiserver            5                   48d1fe9bb2718       kube-apiserver-node
dce5a867daa9e       6f707f569b572       18 minutes ago      Exited              kube-apiserver            4                   48d1fe9bb2718       kube-apiserver-node

# 测试一下,这个webhook只是校验是否指定了镜像的tag,非latest
kubectl run web --image=nginx
Error from server (Forbidden): pods "web" is forbidden: image policy webhook backend denied one or more images: Images using latest tag are not allowed
kubectl run web --image=nginx:1.16
pod/web created
```
