
# kubernetes二次开发

## Kubebuilder实践

### 1、常见的开发框架

* Charmed Operator Framework
* kubebuilder
* KubeOps (.NET operator SDK)
* KUDO (Kubernetes Universal Declarative Operator)
* Metacontroller along with WebHooks that you implement yourself
* Operator Framework
* shell-operator

### 2、kubebuilder安装

> kubebuilder book：<https://book.kubebuilder.io/quick-start.html>
> kubebuilder GitHub：<https://github.com/kubernetes-sigs/kubebuilder>
> kubebuilder安装比较的简单。由于对于windows支持不是很好，建议在Linux上安装。
> 必要条件:
>
> * go version v1.15+ (kubebuilder v3.0 < v3.1).
> * go version v1.16+ (kubebuilder v3.1 < v3.3).
> * go version v1.17+ (kubebuilder v3.3+).
> * docker version 17.03+.
> * kubectl version v1.11.3+.
> * Access to a Kubernetes v1.11.3+ cluster.

### 查看go 、docker和kubernetes版本

```bash
[root@master ~]# go version
go version go7 linux/amd64
[root@master ~]# kubectl version
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v5", GitCommit:"aea7bbadd2fc0cd689de94a54e5b7b758869d691", GitTreeState:"clean", BuildDate:"2021-09-15T21:10:45Z", GoVersion:"go8", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v5", GitCommit:"aea7bbadd2fc0cd689de94a54e5b7b758869d691", GitTreeState:"clean", BuildDate:"2021-09-15T21:04:16Z", GoVersion:"go8", Compiler:"gc", Platform:"linux/amd64"}
[root@master ~]# docker version
Client: Docker Engine - Community
 Version:           12
 API version:       41
 Go version:        go12
 Git commit:        e91ed57
 Built:             Mon Dec 13 11:45:22 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          12
  API version:      41 (minimum version 12)
  Go version:       go12
  Git commit:       459d0df
  Built:            Mon Dec 13 11:43:44 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          12
  GitCommit:        7b11cfaabd73bb80907dd23182b9347b4245eb5d
 runc:
  Version:          2
  GitCommit:        v2-0-g52b36a2
 docker-init:
  Version:          0
  GitCommit:        de40ad0
[root@master ~]#
```

> 注：
> （1）关于在centos8.2中安装gov1.17，见：centos 8.2中安装go 1.17
> （2）关于极速安装kubernetes，见：借助kubesphere极速安装Kubernetes
> 由于我们的go 版本为1.17，所以应该下载kubebuilder版本v3.3+

### 下载并安装kuberbuilder

```bash
# download kubebuilder and install locally.
$ curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
$ chmod +x kubebuilder && mv kubebuilder /usr/local/bin/
```

### 3、kubebuilder初步实践

### 初始化

```bash
[root@master ~]# mkdir -p $GOPATH/src/kubebuilder-demo
[root@master ~]# cd $GOPATH/src/kubebuilder-demo
```

```bash
[root@master kubebuilder-demo]# kubebuilder init --domain demo.kubebuilder.io
Writing kustomize manifests for you to edit...
Writing scaffold for you to edit...
Get controller runtime:
$ go get sigs.k8s.io/controller-runtime@v0
Update dependencies:
$ go mod tidy
go: downloading github.com/stretchr/testify v0
go: downloading github.com/onsi/gomega v0
go: downloading github.com/onsi/ginkgo v5
go: downloading github.com/go-logr/zapr v0
go: downloading go.uber.org/zap v1
go: downloading github.com/pmezard/go-difflib v0
go: downloading github.com/Azure/go-autorest/autorest v18
go: downloading github.com/Azure/go-autorest v0+incompatible
go: downloading github.com/Azure/go-autorest/autorest/adal v13
go: downloading go.uber.org/goleak v12
go: downloading go.uber.org/atomic v0
go: downloading go.uber.org/multierr v0
go: downloading github.com/benbjohnson/clock v0
go: downloading gopkg.in/check.v1 v0-20200227125254-8fa46927fb4f
go: downloading github.com/Azure/go-autorest/logger v1
go: downloading github.com/Azure/go-autorest/tracing v0
go: downloading github.com/Azure/go-autorest/autorest/mocks v1
go: downloading cloud.google.com/go v0
go: downloading github.com/Azure/go-autorest/autorest/date v0
go: downloading github.com/form3tech-oss/jwt-go v3+incompatible
go: downloading golang.org/x/crypto v0-20210817164053-32db794688a5
go: downloading github.com/nxadm/tail v8
go: downloading github.com/niemeyer/pretty v0-20200227124842-a10e7caefd8e
go: downloading gopkg.in/tomb.v1 v0-20141024135613-dd632973f1e7
go: downloading github.com/kr/text v0
Next: define a resource with:
```

> 注意：如果你不是在$GOPATH/src路径下进行初始化，请添加上--repo参数作为项目MODULE的名称

```bash
mkdir -p ~/projects/guestbook
cd ~/projects/guestbook
kubebuilder init --domain my.domain --repo my.domain/guestbook
 ```
>
> > Developing in $GOPATH
> > If your project is initialized within `GOPATH`, the implicitly called `go mod init` will interpolate the module path for you. Otherwise `--repo=` must be set.
> > Read the Go modules blogpost if unfamiliar with the module system.

### 创建API

```bash
[root@master kubebuilder-demo]# kubebuilder create api --group myapp --version v1 --kind Redis
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing kustomize manifests for you to edit...
Writing scaffold for you to edit...
api/v1/redis_types.go
controllers/redis_controller.go
Update dependencies:
$ go mod tidy
Running make:
$ make generate
go: creating new go.mod: module tmp
Downloading sigs.k8s.io/controller-tools/cmd/controller-gen@v0
go: downloading sigs.k8s.io/controller-tools v0
go: downloading github.com/spf13/cobra v1
go: downloading github.com/gobuffalo/flect v3
go: downloading golang.org/x/tools v6-20210820212750-d4cc65f0b2ff
go: downloading github.com/fatih/color v0
go: downloading github.com/inconshreveable/mousetrap v0
go: downloading github.com/mattn/go-colorable v8
go: downloading github.com/mattn/go-isatty v12
go: downloading github.com/google/go-cmp v6
go: downloading golang.org/x/sys v0-20210831042530-f4d43177bf5e
go: downloading golang.org/x/mod v2
go get: installing executables with 'go get' in module mode is deprecated.
        To adjust and download dependencies of the current module, use 'go get -d'.
        To install using requirements of the current module, use 'go install'.
        To install ignoring the current module, use 'go install' with a version,
        like 'go install example.com/cmd@latest'.
        For more information, see https://golang.org/doc/go-get-install-deprecation
        or run 'go help get' or 'go help install'.
go get: added github.com/fatih/color v0
go get: added github.com/go-logr/logr v0
go get: added github.com/gobuffalo/flect v3
go get: added github.com/gogo/protobuf v2
go get: added github.com/google/go-cmp v6
go get: added github.com/google/gofuzz v0
go get: added github.com/inconshreveable/mousetrap v0
go get: added github.com/json-iterator/go v12
go get: added github.com/mattn/go-colorable v8
go get: added github.com/mattn/go-isatty v12
go get: added github.com/modern-go/concurrent v0-20180306012644-bacd9c7ef1dd
go get: added github.com/modern-go/reflect2 v2
go get: added github.com/spf13/cobra v1
go get: added github.com/spf13/pflag v5
go get: added golang.org/x/mod v2
go get: added golang.org/x/net v0-20210825183410-e898025ed96a
go get: added golang.org/x/sys v0-20210831042530-f4d43177bf5e
go get: added golang.org/x/text v7
go get: added golang.org/x/tools v6-20210820212750-d4cc65f0b2ff
go get: added golang.org/x/xerrors v0-20200804184101-5ec99f83aff1
go get: added gopkg.in/inf.v0 v1
go get: added gopkg.in/yaml.v2 v0
go get: added gopkg.in/yaml.v3 v0-20210107192922-496545a6307b
go get: added k8s.io/api v0
go get: added k8s.io/apiextensions-apiserver v0
go get: added k8s.io/apimachinery v0
go get: added k8s.io/klog/v2 v0
go get: added k8s.io/utils v0-20210930125809-cb0fa318a74b
go get: added sigs.k8s.io/controller-tools v0
go get: added sigs.k8s.io/json v0-20211020170558-c049b76a60c6
go get: added sigs.k8s.io/structured-merge-diff/v4 v2
go get: added sigs.k8s.io/yaml v0
/root/gopath/src/kubebuilder-demo/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
Next: implement your new API and generate the manifests (e.g. CRDs,CRs) with:
$ make manifests
[root@master kubebuilder-demo]#
```

### make install

```bash
[root@master kubebuilder-demo]# make install
/root/gopath/src/kubebuilder-demo/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
go: creating new go.mod: module tmp
Downloading sigs.k8s.io/kustomize/kustomize/v3@v7
go: downloading sigs.k8s.io/kustomize/kustomize/v3 v7
go: downloading k8s.io/client-go v10
go: downloading github.com/spf13/cobra v0
go: downloading sigs.k8s.io/kustomize/api v5
go: downloading sigs.k8s.io/kustomize/cmd/config v5
go: downloading github.com/evanphx/json-patch v0+incompatible
go: downloading k8s.io/apimachinery v10
go: downloading sigs.k8s.io/kustomize/kyaml v4
go: downloading github.com/gogo/protobuf v1
go: downloading k8s.io/klog v0
go: downloading sigs.k8s.io/structured-merge-diff/v3 v0
go: downloading sigs.k8s.io/structured-merge-diff v0-20190525122527-15d366b2352e
go: downloading k8s.io/kube-openapi v0-20200410145947-61e04a5be9a6
go: downloading k8s.io/api v10
go: downloading github.com/olekukonko/tablewriter v4
go: downloading github.com/go-errors/errors v1
go: downloading gopkg.in/yaml.v2 v0
go: downloading github.com/yujunz/go-getter v1-lite
go: downloading github.com/json-iterator/go v8
go: downloading github.com/googleapis/gnostic v0
go: downloading github.com/Azure/go-autorest/autorest v0
go: downloading github.com/Azure/go-autorest/autorest/adal v0
go: downloading golang.org/x/oauth2 v0-20190604053449-0f29369cfe45
go: downloading github.com/gophercloud/gophercloud v0
go: downloading gopkg.in/yaml.v3 v0-20200313102051-9f266ea9e77c
go: downloading github.com/monochromegane/go-gitignore v0-20200626010858-205db1a8cc00
go: downloading github.com/stretchr/testify v1
go: downloading github.com/xlab/treeprint v0-20181112141820-a009c3971eca
go: downloading github.com/go-openapi/strfmt v5
go: downloading github.com/go-openapi/validate v8
go: downloading github.com/mattn/go-runewidth v7
go: downloading github.com/hashicorp/go-multierror v0
go: downloading golang.org/x/net v0-20200625001655-4c5254603344
go: downloading github.com/bgentry/go-netrc v0-20140422174119-9fd32a8b3d3d
go: downloading github.com/hashicorp/go-cleanhttp v0
go: downloading github.com/hashicorp/go-safetemp v0
go: downloading github.com/hashicorp/go-version v0
go: downloading github.com/mitchellh/go-homedir v0
go: downloading github.com/mitchellh/go-testing-interface v0
go: downloading github.com/ulikunitz/xz v5
go: downloading github.com/Azure/go-autorest/logger v0
go: downloading github.com/Azure/go-autorest/tracing v0
go: downloading github.com/Azure/go-autorest/autorest/date v0
go: downloading github.com/dgrijalva/jwt-go v0+incompatible
go: downloading cloud.google.com/go v0
go: downloading google.golang.org/appengine v0
go: downloading github.com/qri-io/starlib v2-20200213133954-ff2e8cd5ef8d
go: downloading go.starlark.net v0-20200306205701-8dd3e2ee1dd5
go: downloading github.com/asaskevich/govalidator v0-20190424111038-f61b66f89f4a
go: downloading github.com/go-openapi/errors v2
go: downloading github.com/mitchellh/mapstructure v2
go: downloading go.mongodb.org/mongo-driver v2
go: downloading github.com/golang/protobuf v2
go: downloading github.com/google/shlex v0-20191202100458-e7afc7fbc510
go: downloading github.com/hashicorp/errwrap v0
go: downloading github.com/go-openapi/analysis v5
go: downloading github.com/go-openapi/loads v4
go: downloading github.com/go-openapi/runtime v4
go: downloading golang.org/x/crypto v0-20200622213623-75b288015ac9
go: downloading golang.org/x/time v0-20190308202827-9d24e82272b4
go: downloading k8s.io/utils v0-20200324210504-a9aa75ae1b89
go: downloading golang.org/x/text v2
go: downloading github.com/emicklei/go-restful v0-20170410110728-ff4f55a20633
go: downloading golang.org/x/sys v0-20200323222414-85ca7c5b95cd
go: downloading github.com/go-stack/stack v0
go get: installing executables with 'go get' in module mode is deprecated.
        To adjust and download dependencies of the current module, use 'go get -d'.
        To install using requirements of the current module, use 'go install'.
        To install ignoring the current module, use 'go install' with a version,
        like 'go install example.com/cmd@latest'.
        For more information, see https://golang.org/doc/go-get-install-deprecation
        or run 'go help get' or 'go help install'.
go get: added cloud.google.com/go v0
go get: added github.com/Azure/go-autorest/autorest v0
go get: added github.com/Azure/go-autorest/autorest/adal v0
go get: added github.com/Azure/go-autorest/autorest/date v0
go get: added github.com/Azure/go-autorest/logger v0
go get: added github.com/Azure/go-autorest/tracing v0
go get: added github.com/PuerkitoBio/purell v1
go get: added github.com/PuerkitoBio/urlesc v0-20170810143723-de5bf2ad4578
go get: added github.com/asaskevich/govalidator v0-20190424111038-f61b66f89f4a
go get: added github.com/bgentry/go-netrc v0-20140422174119-9fd32a8b3d3d
go get: added github.com/davecgh/go-spew v1
go get: added github.com/dgrijalva/jwt-go v0+incompatible
go get: added github.com/emicklei/go-restful v0-20170410110728-ff4f55a20633
go get: added github.com/evanphx/json-patch v0+incompatible
go get: added github.com/go-errors/errors v1
go get: added github.com/go-openapi/analysis v5
go get: added github.com/go-openapi/errors v2
go get: added github.com/go-openapi/jsonpointer v3
go get: added github.com/go-openapi/jsonreference v3
go get: added github.com/go-openapi/loads v4
go get: added github.com/go-openapi/runtime v4
go get: added github.com/go-openapi/spec v5
go get: added github.com/go-openapi/strfmt v5
go get: added github.com/go-openapi/swag v5
go get: added github.com/go-openapi/validate v8
go get: added github.com/go-stack/stack v0
go get: added github.com/gogo/protobuf v1
go get: added github.com/golang/protobuf v2
go get: added github.com/google/gofuzz v0
go get: added github.com/google/shlex v0-20191202100458-e7afc7fbc510
go get: added github.com/googleapis/gnostic v0
go get: added github.com/gophercloud/gophercloud v0
go get: added github.com/hashicorp/errwrap v0
go get: added github.com/hashicorp/go-cleanhttp v0
go get: added github.com/hashicorp/go-multierror v0
go get: added github.com/hashicorp/go-safetemp v0
go get: added github.com/hashicorp/go-version v0
go get: added github.com/inconshreveable/mousetrap v0
go get: added github.com/json-iterator/go v8
go get: added github.com/mailru/easyjson v0
go get: added github.com/mattn/go-runewidth v7
go get: added github.com/mitchellh/go-homedir v0
go get: added github.com/mitchellh/go-testing-interface v0
go get: added github.com/mitchellh/mapstructure v2
go get: added github.com/modern-go/concurrent v0-20180306012644-bacd9c7ef1dd
go get: added github.com/modern-go/reflect2 v1
go get: added github.com/monochromegane/go-gitignore v0-20200626010858-205db1a8cc00
go get: added github.com/olekukonko/tablewriter v4
go get: added github.com/pkg/errors v1
go get: added github.com/pmezard/go-difflib v0
go get: added github.com/qri-io/starlib v2-20200213133954-ff2e8cd5ef8d
go get: added github.com/spf13/cobra v0
go get: added github.com/spf13/pflag v5
go get: added github.com/stretchr/testify v1
go get: added github.com/ulikunitz/xz v5
go get: added github.com/xlab/treeprint v0-20181112141820-a009c3971eca
go get: added github.com/yujunz/go-getter v1-lite
go get: added go.mongodb.org/mongo-driver v2
go get: added go.starlark.net v0-20200306205701-8dd3e2ee1dd5
go get: added golang.org/x/crypto v0-20200622213623-75b288015ac9
go get: added golang.org/x/net v0-20200625001655-4c5254603344
go get: added golang.org/x/oauth2 v0-20190604053449-0f29369cfe45
go get: added golang.org/x/sys v0-20200323222414-85ca7c5b95cd
go get: added golang.org/x/text v2
go get: added golang.org/x/time v0-20190308202827-9d24e82272b4
go get: added google.golang.org/appengine v0
go get: added gopkg.in/inf.v0 v1
go get: added gopkg.in/yaml.v2 v0
go get: added gopkg.in/yaml.v3 v0-20200313102051-9f266ea9e77c
go get: added k8s.io/api v10
go get: added k8s.io/apimachinery v10
go get: added k8s.io/client-go v10
go get: added k8s.io/klog v0
go get: added k8s.io/kube-openapi v0-20200410145947-61e04a5be9a6
go get: added k8s.io/utils v0-20200324210504-a9aa75ae1b89
go get: added sigs.k8s.io/kustomize/api v5
go get: added sigs.k8s.io/kustomize/cmd/config v5
go get: added sigs.k8s.io/kustomize/kustomize/v3 v7
go get: added sigs.k8s.io/kustomize/kyaml v4
go get: added sigs.k8s.io/structured-merge-diff/v3 v0
go get: added sigs.k8s.io/yaml v0
/root/gopath/src/kubebuilder-demo/bin/kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/redis.myapp.demo.kubebuilder.io created
[root@master kubebuilder-demo]
```

### 查看所创建的CRD

```bash
[root@master kubebuilder-demo]# kubectl get crds |grep redis
redis.myapp.demo.kubebuilder.io                       2022-02-13T11:32:20Z
[root@master kubebuilder-demo]#
```

### 4、业务开发

> 本次实践中，我们将创建自定义资源Redis，实现的功能
> （1）创建CRD
> （2）创建Kind ：Redis
> （3）创建POD，支持副本数控制，验证
> （3）创建多副本POD，支持副本收缩
> 为了开发方便，可以将上面的生成的kubebuilder-demo文件拷贝到windows下，并导入到goland中。并配置SFTP自动上传到$GOPATH/src/kubebuilder-demo下
> ![img](https://img2022.cnblogs.com/blog/1722983/202202/1722983-20220214221457761-1988314662.png)
> ![img](https://img2022.cnblogs.com/blog/1722983/202202/1722983-20220214221722384-235338823.png)
> ![img](https://img2022.cnblogs.com/blog/1722983/202202/1722983-20220214221818973-425192864.png)
> 另外也可以直接将文件拷贝到本地后，直接在本地运行

* （1）将Kubernetes的~/.kube/config文件，拷贝到windows的$ %HOME% /.kube/下  
    如：C:\\Users\\Administrator\.kube，之所以要这个做是因为kubebuilder中加载kubeconfig文件的顺序所决定的。详情见：Kubebuilder认证配置文件的加载
* （2）针对于_types.go文件的修改，需要重新编译生成“zz_generated.deepcopy.go”文件，可以在Linux上make install后再拷贝回来

### 验证

> Kubebuilder中的验证是通过标签的方式来完成，标签的基本形式为：//+some-tag 或//+some-other-tag=value
> 标签分为两种：

* 声明在package行上的全局标签
* 在类型声明前的局部标签（如在一个struct定义前）

> `（1）redis_types.go，修改Foo为Port`

```go
    // +kubebuilder:validation:Maximum:=6380
	// +kubebuilder:validation:Minimum:=6370

	Port int `json:"port,omitempty"`
```

> ![img](https://img2022.cnblogs.com/blog/1722983/202202/1722983-20220214222123032-347003838.png)
> 这里我们只是针对于port的范围做验证，有关Kubernetes的验证见：<https://book.kubebuilder.io/reference/markers/crd-validation.html>
> `（2）修改myapp_v1_redis.yaml文件，增加端口字段：`

```yaml
apiVersion: myapp.demo.kubebuilder.io/v1
kind: Redis
metadata:
  name: redis-sample
spec:
  # TODO(user): Add fields here
  port: 6379
```

> 由于这里我们修改了Redis struct的字段定义，所以需要重新make install
> 此时，我们可以看到自己的CRD中关于port字段的验证规则：

```bash
[root@master ~]# kubectl get crds |grep redis
redis.myapp.demo.kubebuilder.io                       2022-02-13T11:32:20Z
[root@master ~]# kubectl get crds/redis.myapp.demo.kubebuilder.io  -oyaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    controller-gen.kubebuilder.io/version: v0
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apiextensions.k8s.io/v1","kind":"CustomResourceDefinition","metadata":{"annotations":{"controller-gen.kubebuilder.io/version":"v0"},"creationTimestamp":null,"name":"redis.myapp.demo.kubebuilder.io"},"spec":{"group":"myapp.demo.kubebuilder.io","names":{"kind":"Redis","listKind":"RedisList","plural":"redis","singular":"redis"},"scope":"Namespaced","versions":[{"name":"v1","schema":{"openAPIV3Schema":{"description":"Redis is the Schema for the redis API","properties":{"apiVersion":{"description":"APIVersion defines the versioned schema of this representation of an object. Servers should convert recognized schemas to the latest internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources","type":"string"},"kind":{"description":"Kind is a string value representing the REST resource this object represents. Servers may infer this from the endpoint the client submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds","type":"string"},"metadata":{"type":"object"},"spec":{"description":"RedisSpec defines the desired state of Redis","properties":{"num":{"type":"integer"},"port":{"maximum":6380,"minimum":6370,"type":"integer"}},"type":"object"},"status":{"description":"RedisStatus defines the observed state of Redis","type":"object"}},"type":"object"}},"served":true,"storage":true,"subresources":{"status":{}}}]},"status":{"acceptedNames":{"kind":"","plural":""},"conditions":[],"storedVersions":[]}}
  creationTimestamp: "2022-02-13T11:32:20Z"
  generation: 4
  name: redis.myapp.demo.kubebuilder.io
  resourceVersion: "510294"
  uid: 91122c22-e52e-46cb-bcde-0a5b20578bc2
spec:
  conversion:
    strategy: None
  group: myapp.demo.kubebuilder.io
  names:
    kind: Redis
    listKind: RedisList
    plural: redis
    singular: redis
  scope: Namespaced
  versions:
  - name: v1
    schema:
      openAPIV3Schema:
        description: Redis is the Schema for the redis API
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: RedisSpec defines the desired state of Redis
            properties:
              num:
                type: integer
              port:
                maximum: 6380 #验证规则
                minimum: 6370
                type: integer
            type: object
          status:
            description: RedisStatus defines the observed state of Redis
            type: object
        type: object
    served: true
    storage: true
    subresources:
      status: {}
status:
  acceptedNames:
    kind: Redis
    listKind: RedisList
    plural: redis
    singular: redis
  conditions:
  - lastTransitionTime: "2022-02-13T11:32:20Z"
    message: no conflicts found
    reason: NoConflicts
    status: "True"
    type: NamesAccepted
  - lastTransitionTime: "2022-02-13T11:32:20Z"
    message: the initial names have been accepted
    reason: InitialNamesAccepted
    status: "True"
    type: Established
  storedVersions:
  - v1
[root@master ~]
```

> `（3）应用myapp_v1_redis.yaml文件，创建Kind Redis`

```bash
[root@master kubebuilder-demo]# kubectl apply -f  config/samples/myapp_v1_redis.yaml 
redis.myapp.demo.kubebuilder.io/redis-sample configured

[root@master ~]# kubectl get Redis
NAME           AGE
redis-sample   95m
```

> 上面就显示了NAME和AGE，实际上这里的显示格式，也是可以自定义的。
> `（4）修改myapp_v1_redis.yaml的port为6382，能够看到报错`

```bash
[root@master kubebuilder-demo]# kubectl apply -f  config/samples/myapp_v1_redis.yaml 
The Redis "redis-sample" is invalid: spec.port: Invalid value: 6382: spec.port in body should be less than or equal to 6380
```

### 创建POD

> 创建controllers/redis_helper.go文件：

```go
package controllers

import (
	"context"
	coreV1 "k8s.io/api/core/v1"
	v1 "kubebuilder-demo/api/v1"
	"sigs.k8s.io/controller-runtime/pkg/client"
)

func CreateRedis(client client.Client, redisConfig *vRedis) error {
	newPod := &coreVPod{}
	newPod.Name = redisConfig.Name
	newPod.Namespace = redisConfig.Namespace
	newPod.Spec.Containers = []coreVContainer{
		{
			Name:            redisConfig.Name,
			Image:           "redis:5-alpine",
			ImagePullPolicy: coreVPullIfNotPresent,
			Ports: []coreVContainerPort{
				{
					ContainerPort: int32(redisConfig.Spec.Port),
				},
			},
		},
	}
	return client.Create(context.Background(), newPod)
}
```

> 修改controllers/redis_controller.go文件的Reconcile，增加如下逻辑：

```go
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here
	redis := &myappvRedis{}
	err := r.Get(ctx, req.NamespacedName, redis)
	if err != nil {
		fmt.Println("error:", err)
	} else {
		fmt.Println("redis Obj:", redis)
		err := CreateRedis(r.Client, redis)
		fmt.Println("create pod failue,", err)
		return ctrl.Result{}, err
	}

	return ctrl.Result{}, nil
}
```

> 此时make run，并打开另外一个窗口执行kubectl apply -f config/samples/myapp_v1_redis.yaml
> 注意：为了不影响结果，需要删除前面一个步骤生成的Kind；kubectl delete -f

```bash
[root@master kubebuilder-demo]# kubectl get Redis 
No resources found in default namespace.
[root@master kubebuilder-demo]# kubectl get Redis
No resources found in default namespace.
[root@master kubebuilder-demo]# kubectl apply -f config/samples/myapp_v1_redis.yaml 
redis.myapp.demo.kubebuilder.io/redis-sample created
[root@master kubebuilder-demo]# kubectl get Redis 
NAME           AGE
redis-sample   21s
[root@master kubebuilder-demo]# kubectl get pods #已经创建了pod
NAME           READY   STATUS    RESTARTS   AGE
redis-sample   1/1     Running   0          16s
[root@master kubebuilder-demo]#
```

> 注意：这里存在一个问题，如果我们将创建资源的CRD文件删掉，则POD依旧存在：

```bash
[root@master kubebuilder-demo]# kubectl delete -f config/samples/myapp_v1_redis.yaml 
redis.myapp.demo.kubebuilder.io "redis-sample" deleted
[root@master kubebuilder-demo]# kubectl get pods #pod依旧存在
NAME           READY   STATUS    RESTARTS   AGE
redis-sample   1/1     Running   0          72s
[root@master kubebuilder-demo]# kubectl get Redis
No resources found in default namespace.
[root@master kubebuilder-demo]#
```

> 这就造成了所创建POD的孤立存在。
> 为此，我们可以使用finalizers来限制，如果POD没有删除，不能删除对应POD。
> 有关finalizers的使用见：<https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/>
> 另外需要说明的是，POD究竟是如何被创建出来的，追踪代码能够发现实际上调用的就是Kubernetes的API：
> <https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#create-pod-v1-core>
> HTTP Request
>
```text
POST /api/v1/namespaces/{namespace}/pods
```
>
> | Parameter | Description |
> | --- | --- |
> | `namespace` | object name and auth scope, such as for teams and projects |
>
> ### Query Parameters
>
> | Parameter | Description |
> | --- | --- |
> | `pretty` | If 'true', then the output is pretty printed. |
> | `dryRun` | When present, indicates that modifications should not be persisted. An invalid or unrecognized dryRun directive will result in an error response and no further processing of the request. Valid values are: - All: all dry run stages will be processed |
> | `fieldManager` | fieldManager is a name associated with the actor or entity that is making these changes. The value must be less than or 128 characters long, and only contain printable characters, as defined by <https://golang.org/pkg/unicode/#IsPrint>. |
>
> ### Body Parameters
>
> | Parameter | Description |
> | --- | --- |
> | `body` _Pod_ |  |
>
> ### Response
>
> | Code | Description |
> | --- | --- |
> | 200 _Pod_ | OK |
> | 201 _Pod_ | Created |
> | 202 _Pod_ | Accepte |
>
> 跟踪代码client.Create(context.Background(), newPod)，
>
> sigs.k8s.io/controller-runtime@v0.11.0/pkg/client/client.go

```go
// Create implements client.Client.
func (c *client) Create(ctx context.Context, obj Object, opts ...CreateOption) error {
	switch obj.(type) {
	case *unstructured.Unstructured:
		return c.unstructuredClient.Create(ctx, obj, opts...)
	case *metavPartialObjectMetadata:
		return fmt.Errorf("cannot create using only metadata")
	default:
		return c.typedClient.Create(ctx, obj, opts...)//
	}
}
```

```go
// Create implements client.Client.
func (c *typedClient) Create(ctx context.Context, obj Object, opts ...CreateOption) error {
	o, err := c.cache.getObjMeta(obj)
	if err != nil {
		return err
	}

	createOpts := &CreateOptions{}
	createOpts.ApplyOptions(opts)
	return o.Post().
		NamespaceIfScoped(o.GetNamespace(), o.isNamespaced()).
		Resource(o.resource()).
		Body(obj).
		VersionedParams(createOpts.AsCreateOptions(), c.paramCodec).
		Do(ctx).
		Into(obj)
}
```

```go
// request connects to the server and invokes the provided function when a server response is
// received. It handles retry behavior and up front validation of requests. It will invoke
// fn at most once. It will return an error if a problem occurred prior to connecting to the
// server - the provided function is responsible for handling server errors.
func (r *Request) request(ctx context.Context, fn func(*http.Request, *http.Response)) error {
	//Metrics for total request latency
	start := time.Now()
	defer func() {
		metrics.RequestLatency.Observe(ctx, r.verb, r.finalURLTemplate(), time.Since(start))
	}()

	if r.err != nil {
		klog.V(4).Infof("Error in request: %v", r.err)
		return r.err
	}

	if err := r.requestPreflightCheck(); err != nil {
		return err
	}

	client := r.c.Client
	if client == nil {
		client = http.DefaultClient
	}

	// Throttle the first try before setting up the timeout configured on the
	// client. We don't want a throttled client to return timeouts to callers
	// before it makes a single request.
	if err := r.tryThrottle(ctx); err != nil {
		return err
	}

	if r.timeout > 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, r.timeout)
		defer cancel()
	}

	// Right now we make about ten retry attempts if we get a Retry-After response.
	var retryAfter *RetryAfter
	for {
		req, err := r.newHTTPRequest(ctx)
		if err != nil {
			return err
		}

		r.backoff.Sleep(r.backoff.CalculateBackoff(r.URL()))
		if retryAfter != nil {
			// We are retrying the request that we already send to apiserver
			// at least once before.
			// This request should also be throttled with the client-internal rate limiter.
			if err := r.tryThrottleWithInfo(ctx, retryAfter.Reason); err != nil {
				return err
			}
			retryAfter = nil
		}
		resp, err := client.Do(req) //在这里调用http.client发送请求报文
		updateURLMetrics(ctx, r, resp, err)
		....
    } 
}
```

> 最终在这里完成发送请求，将POD放入到请求荷载中：
>
> net/http/client.go:725
>
> ![img](https://img2022.cnblogs.com/blog/1722983/202202/1722983-20220217215028037-1816335387.png)

### 创建多副本的POD

> `（1）修改api/v1/redis_types.go文件，增加副本Num字段：`

```go
// RedisSpec defines the desired state of Redis
type RedisSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of Redis. Edit redis_types.go to remove/update

	// +kubebuilder:validation:Maximum:=6380
	// +kubebuilder:validation:Minimum:=6370

	Port int `json:"port,omitempty"`

	Num int `json:"num,omitempty"`
}
```

> `（2）修改controllers/redis_helper.go：`

```go
package controllers

import (
	"context"
	"fmt"
	coreV1 "k8s.io/api/core/v1"
	v1 "kubebuilder-demo/api/v1"
	"sigs.k8s.io/controller-runtime/pkg/client"
)
//判断对应名称的pod在finalizer中是否存在
func isExist(podName string, redis *vRedis) bool {
	for _, finalizer := range redis.Finalizers {
		if podName == finalizer {
			return true
		}
	}
	return false
}
//实际创建POD
func CreateRedis(podName string, client client.Client, redisConfig *vRedis) (string, error) {
	if isExist(podName, redisConfig) {
		return "", nil
	}
	newPod := &coreVPod{}
	newPod.Name = podName
	newPod.Namespace = redisConfig.Namespace
	newPod.Spec.Containers = []coreVContainer{
		{
			Name:            redisConfig.Name,
			Image:           "redis:5-alpine",
			ImagePullPolicy: coreVPullIfNotPresent,
			Ports: []coreVContainerPort{
				{
					ContainerPort: int32(redisConfig.Spec.Port),
				},
			},
		},
	}
	return podName, client.Create(context.Background(), newPod)
}
//为多副本POD命名
func GetRedisPodNames(redis *vRedis) []string {
	podNames := make([]string, redis.Spec.Num)
	for i := 0; i < redis.Spec.Num; i++ {
		podNames[i] = fmt.Sprintf("%s-%d", redis.Name, i)
	}

	return podNames
}
```

> `（3）修改controllers/redis_controller.go：`

```go
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0/pkg/reconcile
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here
	redis := &myappvRedis{}
	err := r.Get(ctx, req.NamespacedName, redis)
	if err != nil {
		return ctrl.Result{}, err
	}

	fmt.Println("redis Obj:", redis)
    //取得需要创建的POD副本的名称
	podNames := GetRedisPodNames(redis)
	fmt.Println("podNames,", podNames)
	updateFlag := false

    //删除POD，删除Kind:Redis过程中，会自动加上DeletionTimestamp字段，据此判断是否删除了自定义资源
	if !redis.DeletionTimestamp.IsZero() {
		return ctrl.Result{}, r.clearRedis(ctx, redis)
	}

	//创建POD
	for _, podName := range podNames {
		finalizerPodName, err := CreateRedis(podName, r.Client, redis)
		if err != nil {
			fmt.Println("create pod failue,", err)
			return ctrl.Result{}, err
		}
		if finalizerPodName == "" {
			continue
		}
        //若该pod已经不在finalizers中，则添加
		redis.Finalizers = append(redis.Finalizers, finalizerPodName)
		updateFlag = true
	}
	//更新Kind Redis状态
	if updateFlag {
		err = r.Client.Update(ctx, redis)
		if err != nil {
			return ctrl.Result{}, err
		}
	}
	return ctrl.Result{}, nil
}
//删除POD逻辑
func (r *RedisReconciler) clearRedis(ctx context.Context, redis *myappvRedis) error {
    //从finalizers中取出podName，然后执行删除
	for _, finalizer := range redis.Finalizers {
        //删除pod
		err := r.Client.Delete(ctx, &vPod{
			ObjectMeta: metavObjectMeta{
				Name:      finalizer,
				Namespace: redis.Namespace,
			},
		})
		if err != nil {
			return err
		}
	}
    //清空finializers，只要它有值，就无法删除Kind
	redis.Finalizers = []string{}
	return r.Client.Update(ctx, redis)
}
```

> `（4）修改config/samples/myapp_v1_redis.yaml，增加num`

```yaml
apiVersion: myapp.demo.kubebuilder.io/v1
kind: Redis
metadata:
  name: redis-sample
spec:
  # TODO(user): Add fields here
  port: 6379
  num: 2
```

> `（5）make install后make run，在另外一个窗口中执行kubectl apply -f`
>
> 注意：每次修改type类型定义文件，都需要进行make install

```bash
[root@master kubebuilder-demo]# kubectl apply -f  config/samples/myapp_v1_redis.yaml 
redis.myapp.demo.kubebuilder.io/redis-sample created
[root@master kubebuilder-demo]# kubectl get Redis
NAME           AGE
redis-sample   6s
[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
redis-sample-0   1/1     Running   0          10s
redis-sample-1   1/1     Running   0          10s
[root@master kubebuilder-demo]#
```

> 此时查看Kind:Redis的yaml文件，能够看到finalizers字段上填充了两个POD的名字

```bash
[root@master kubebuilder-demo]# kubectl get Redis/redis-sample -oyaml 
apiVersion: myapp.demo.kubebuilder.io/v1
kind: Redis
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"myapp.demo.kubebuilder.io/v1","kind":"Redis","metadata":{"annotations":{},"name":"redis-sample","namespace":"default"},"spec":{"num":2,"port":6379}}
  creationTimestamp: "2022-02-14T15:13:09Z"
  finalizers:  #字段填充了POD名称
  - redis-sample-0
  - redis-sample-1
  generation: 1
  name: redis-sample
  namespace: default
  resourceVersion: "561478"
  uid: 46a36e7c-2af9-47b8-a882-1a9c11ab7110
spec:
  num: 2
  port: 6379
```

> `（6）此时尝试删除Kind：Redis，对应的POD也会被删除`

```bash
[root@master kubebuilder-demo]# kubectl delete -f  config/samples/myapp_v1_redis.yaml 
redis.myapp.demo.kubebuilder.io "redis-sample" deleted
[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS        RESTARTS   AGE
redis-sample-0   0/1     Terminating   0          88s
redis-sample-1   0/1     Terminating   0          88s
[root@master kubebuilder-demo]# kubectl get Redis
No resources found in default namespace.
[root@master kubebuilder-demo]
```

> `（7）修改config/samples/myapp_v1_redis.yaml的num为1，并再次应用`
>
> 查看POD，能够发现POD数，并未发生变化：

```bash
[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
redis-sample-0   1/1     Running   0          4m27s
redis-sample-1   1/1     Running   0          4m27s
[root@master kubebuilder-demo]#
```

### 副本收缩

> `（1）修改controllers/redis_controller.go，优化副本删除逻辑：`

```go
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here
	redis := &myappvRedis{}
	err := r.Get(ctx, req.NamespacedName, redis)
	if err != nil {
		return ctrl.Result{}, err
	}

	fmt.Println("redis Obj:", redis)

	//（1）删除Kind Redis时，删除所创建的POD
	//（2）副本收缩时，缩减POD
	if !redis.DeletionTimestamp.IsZero() || len(redis.Finalizers) > redis.Spec.Num {
		return ctrl.Result{}, r.clearRedis(ctx, redis)
	}

	//创建POD
	podNames := GetRedisPodNames(redis)
	fmt.Println("podNames,", podNames)
	updateFlag := false
	for _, podName := range podNames {
		finalizerPodName, err := CreateRedis(podName, r.Client, redis)
		if err != nil {
			fmt.Println("create pod failue,", err)
			return ctrl.Result{}, err
		}
		if finalizerPodName == "" {
			continue
		}
		redis.Finalizers = append(redis.Finalizers, finalizerPodName)
		updateFlag = true
	}
	//更新Kind Redis状态
	if updateFlag {
		err = r.Client.Update(ctx, redis)
		if err != nil {
			return ctrl.Result{}, err
		}
	}
	return ctrl.Result{}, nil
}

func (r *RedisReconciler) clearRedis(ctx context.Context, redis *myappvRedis) error {

	//副本数和当前的num数量的差值，要删除的就是这个差值部分，
	//如果两则相等，则删除全部
	//情形如下：
	//（1）finalizers > num  可能出现，删除差值部分
	//（2）finalizers = num 可能出现，删除全部
	//（3）finalizers < num 不可能出现
	var deletedPodNames []string

	//后项删除，即：删除finalizers切片的最后的指定元素
	position := redis.Spec.Num
	if (len(redis.Finalizers) - redis.Spec.Num) != 0 {
		deletedPodNames = redis.Finalizers[position:]
		redis.Finalizers = redis.Finalizers[:position]
	} else {
		deletedPodNames = redis.Finalizers[:]
		redis.Finalizers = []string{}
	}

	fmt.Println("deletedPodNames", deletedPodNames)
	fmt.Println("redis.Finalizers", redis.Finalizers)
	for _, finalizer := range deletedPodNames {
		err := r.Client.Delete(ctx, &vPod{
			ObjectMeta: metavObjectMeta{
				Name:      finalizer,
				Namespace: redis.Namespace,
			},
		})
		if err != nil {
			return err
		}
	}
	return r.Client.Update(ctx, redis)
}
```

> `（2）执行make run，修改config/samples/myapp_v1_redis.yaml的num值为5，创建5个副本的POD`

```bash
[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
redis-sample-0   1/1     Running   0          4m27s
redis-sample-1   1/1     Running   0          4m27s
[root@master kubebuilder-demo]# vi config/samples/myapp_v1_redis.yaml 
[root@master kubebuilder-demo]# kubectl apply -f  config/samples/myapp_v1_redis.yaml 
redis.myapp.demo.kubebuilder.io/redis-sample configured
[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS              RESTARTS   AGE
redis-sample-0   1/1     Running             0          12m
redis-sample-1   1/1     Running             0          12m
redis-sample-2   0/1     ContainerCreating   0          2s
redis-sample-3   0/1     ContainerCreating   0          2s
redis-sample-4   0/1     ContainerCreating   0          2s
[root@master kubebuilder-demo]#
```

> 等待POD创建就绪后，再次修改num为2，查看：

```bash
[root@master kubebuilder-demo]# kubectl apply -f  config/samples/myapp_v1_redis.yaml 
redis.myapp.demo.kubebuilder.io/redis-sample configured
[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS        RESTARTS   AGE
redis-sample-0   1/1     Running       0          14m
redis-sample-1   1/1     Running       0          14m
redis-sample-2   0/1     Terminating   0          2m5s
redis-sample-3   0/1     Terminating   0          2m5s
redis-sample-4   0/1     Terminating   0          2m5s
[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
redis-sample-0   1/1     Running   0          15m
redis-sample-1   1/1     Running   0          15m
[root@master kubebuilder-demo]#
```

> 删除kind:Redis自定义资源：

```bash
[root@master kubebuilder-demo]# kubectl delete -f  config/samples/myapp_v1_redis.yaml 
redis.myapp.demo.kubebuilder.io "redis-sample" deleted
[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS        RESTARTS   AGE
redis-sample-0   0/1     Terminating   0          16m
redis-sample-1   0/1     Terminating   0          16m
[root@master kubebuilder-demo]# kubectl get pods
No resources found in default namespace.
[root@master kubebuilder-demo]# kubectl get Redis
No resources found in default namespace.
[root@master kubebuilder-demo]#
```

### POD删除和重建

> 在前面的POD创建时，我们没有使用Deployment来管理POD，一旦有些POD被删除掉，则不会自动创建。这里我们借助于Watches来实现，同时引入Owner Referenc概念，从中取出Kind和Name，据此来判断删除的是我们POD，然后在等待队列中，增加reconcile.Request，以此来实现对于副本数量的控制。
>
> 关于Owner Reference：
>
> > <https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/>
>
> ### Owner References
>
> > Owner references describe how groups of objects are related. They are properties on resources that specify the relationship to one another, so entire trees of resources can be deleted.
> > Finalizer rules are processed when there are owner references. An owner reference consists of a name and a UID. Owner references link resources within the same namespace, and it also needs a UID for that reference to work. Pods typically have owner references to the owning replica set. So, when deployments or stateful sets are deleted, then the child replica sets and pods are deleted in the process.
>
> `（1）修改controllers/redis_controller.go，监听POD：`

```go
//POD删除时的回调
func (r *RedisReconciler) poddeleteHandler(event event.DeleteEvent, limitingInterface workqueue.RateLimitingInterface) {
	fmt.Println("deleted pod name is :", event.Object.GetName())
    //取得OwnerReference，如果Kind和Name匹配，则触发reconcile.Request，并加入到等待队列
	for _, ownerReference := range event.Object.GetOwnerReferences() {
		if ownerReference.Kind == "Redis" && ownerReference.Name == "redis-sample" {
			limitingInterface.Add(reconcile.Request{
				NamespacedName: types.NamespacedName{
					Namespace: event.Object.GetNamespace(),
					Name:      ownerReference.Name,
				},
			})
		}
	}
}

// SetupWithManager sets up the controller with the Manager.
func (r *RedisReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&myappvRedis{}).
		Watches(&source.Kind{ 
			Type: &vPod{},
		}, handler.Funcs{
			DeleteFunc: r.poddeleteHandler,
		}).
		Complete(r)

}
```

> `（2）修改controllers/redis_helper.go`

```go
//优化判断POD存在逻辑
func isExistPod(podName string, redis *vRedis, client client.Client) bool {
	err := client.Get(context.Background(), types.NamespacedName{
		Name:      podName,
		Namespace: redis.Namespace,
	}, &coreVPod{})

	if err != nil {
		return false
	}
	return true
}

func CreateRedis(podName string, client client.Client, redisConfig *vRedis, scheme *runtime.Scheme) (string, error) {
	if isExistPod(podName, redisConfig, client) {
		return podName, nil
	}
	newPod := &coreVPod{}
	newPod.Name = podName
	newPod.Namespace = redisConfig.Namespace
	newPod.Spec.Containers = []coreVContainer{
		{
			Name:            redisConfig.Name,
			Image:           "redis:5-alpine",
			ImagePullPolicy: coreVPullIfNotPresent,
			Ports: []coreVContainerPort{
				{
					ContainerPort: int32(redisConfig.Spec.Port),
				},
			},
		},
	}
	err := controllerutil.SetControllerReference(redisConfig, newPod, scheme) //为POD增加OwnerReference
	if err != nil {
		return podName, err
	}
	return podName, client.Create(context.Background(), newPod)
}
```

> `（3）Make run`

```bash
#删除旧的自定义资源
[root@master kubebuilder-demo]# kubectl delete -f  config/samples/myapp_v1_redis.yaml

[root@master kubebuilder-demo]# kubectl apply  -f  config/samples/myapp_v1_redis.yaml 
redis.myapp.demo.kubebuilder.io/redis-sample created
[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
redis-sample-0   1/1     Running   0          13s
redis-sample-1   1/1     Running   0          13s
[root@master kubebuilder-demo]# kubectl delete pod redis-sample-1
pod "redis-sample-1" deleted
#在另外一个窗口查看pod情况
[root@master ~]# kubectl get pods
NAME             READY   STATUS        RESTARTS   AGE
redis-sample-0   1/1     Running       0          19s
redis-sample-1   0/1     Terminating   0          90s
[root@master ~]
#pod又被重新创建
[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
redis-sample-0   1/1     Running   0          37s
redis-sample-1   1/1     Running   0          8s
[root@master kubebuilder-demo]#
```

> 查看POD的yaml，观察OwnerReference字段：

```bash
[root@master kubebuilder-demo]# kubectl get pod redis-sample-0 -oyaml 
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/containerID: 00e0b8f104de05271f3b0b9aa2f72e8bf6e919ce5139da10ad1c76e215fd4365
    cni.projectcalico.org/podIP: 264/32
    cni.projectcalico.org/podIPs: 264/32
  creationTimestamp: "2022-02-15T09:11:21Z"
  name: redis-sample-0
  namespace: default
  ownerReferences:
  - apiVersion: myapp.demo.kubebuilder.io/v1
    blockOwnerDeletion: true
    controller: true
    kind: Redis
    name: redis-sample
    uid: ed99405e-df1e-4d24-90f8-5994aa4c7f41
  resourceVersion: "639304"
  uid: 3173af40-498e-4d79-a006-cebaccdf86a9
spec:
  ...
```

> 其中ownerReferences中 uid: ed99405e-df1e-4d24-90f8-5994aa4c7f41为自定义资源Redis的UID：

```bash
[root@master kubebuilder-demo]# kubectl get Redis -oyaml
apiVersion: v1
items:
- apiVersion: myapp.demo.kubebuilder.io/v1
  kind: Redis
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"myapp.demo.kubebuilder.io/v1","kind":"Redis","metadata":{"annotations":{},"name":"redis-sample","namespace":"default"},"spec":{"num":2,"port":6379}}
    creationTimestamp: "2022-02-15T09:09:38Z"
    finalizers:
    - redis-sample-0
    - redis-sample-1
    generation: 1
    name: redis-sample
    namespace: default
    resourceVersion: "639070"
    uid: ed99405e-df1e-4d24-90f8-5994aa4c7f41
  spec:
    num: 2
    port: 6379
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

### 添加事件支持

> 目前在自定义资源Kind:Redis中Events: ：

```bash
[root@master kubebuilder-demo]# kubectl describe Redis/redis-sample
Name:         redis-sample
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  myapp.demo.kubebuilder.io/v1
Kind:         Redis
Metadata:
  Creation Timestamp:  2022-02-15T09:09:38Z
  Finalizers:
    redis-sample-0
    redis-sample-1
  Generation:  1
  Managed Fields:
    API Version:  myapp.demo.kubebuilder.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:num:
        f:port:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2022-02-15T09:09:38Z
    API Version:  myapp.demo.kubebuilder.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:finalizers:
          .:
          v:"redis-sample-0":
          v:"redis-sample-1":
    Manager:         main
    Operation:       Update
    Time:            2022-02-15T09:09:38Z
  Resource Version:  639070
  UID:               ed99405e-df1e-4d24-90f8-5994aa4c7f41
Spec:
  Num:   2
  Port:  6379
Events:  <none>
[root@master kubebuilder-demo]#
```

> 现在需要增加对于增加和删除POD的事件进行记录。
>
> `（1）修改redis_controller.go`

```go
// RedisReconciler reconciles a Redis object
type RedisReconciler struct {
	client.Client
	Scheme *runtime.Scheme
	EventRecorder record.EventRecorder  //增加事件记录
}

func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	...
	//更新Kind Redis状态
	if updateFlag {
		r.EventRecorder.Event(redis,vEventTypeNormal,"Upgrade","scale replicates")//创建POD时，记录事件
		err = r.Client.Update(ctx, redis)
		if err != nil {
			return ctrl.Result{}, err
		}
	}
	return ctrl.Result{}, nil
}


func (r *RedisReconciler) clearRedis(ctx context.Context, redis *myappvRedis) error {

	...
    r.EventRecorder.Event(redis,vEventTypeNormal,"Updated","scale replicates")//删除POD时，记录事件
	return r.Client.Update(ctx, redis)
}
```

> `（2）修改main.go，创建RedisReconciler时，为EventRecorder属性赋值`

```go
func main() {
	...
	if err = (&controllers.RedisReconciler{
		Client:        mgr.GetClient(),
		Scheme:        mgr.GetScheme(),
		EventRecorder: mgr.GetEventRecorderFor("demo.kubebuilder.io"), //名称任意
	}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "Redis")
		os.Exit(1)
	}
	...
}
```

> `（3）make install & make run ，然后测试效果`

```bash
[root@master kubebuilder-demo]# kubectl apply  -f  config/samples/myapp_v1_redis.yaml 
redis.myapp.demo.kubebuilder.io/redis-sample created
[root@master kubebuilder-demo]# kubectl get Redis
NAME           AGE
redis-sample   29s

[root@master kubebuilder-demo]# kubectl get pods
NAME             READY   STATUS    RESTARTS   AGE
redis-sample-0   1/1     Running   0          80s
redis-sample-1   1/1     Running   0          80s
[root@master kubebuilder-demo]# 

[root@master kubebuilder-demo]# kubectl describe Redis/redis-sample
Name:         redis-sample
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  myapp.demo.kubebuilder.io/v1
Kind:         Redis
Metadata:
  Creation Timestamp:  2022-02-15T10:14:38Z
  Finalizers:
    redis-sample-0
    redis-sample-1
  Generation:  1
  Managed Fields:
    API Version:  myapp.demo.kubebuilder.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:num:
        f:port:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2022-02-15T10:14:38Z
    API Version:  myapp.demo.kubebuilder.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:finalizers:
          .:
          v:"redis-sample-0":
          v:"redis-sample-1":
    Manager:         main
    Operation:       Update
    Time:            2022-02-15T10:14:38Z
  Resource Version:  643906
  UID:               aae17d51-0889-40d0-838d-d6f18d411956
Spec:
  Num:   2
  Port:  6379
Events:
  Type    Reason   Age   From                 Message
  ----    ------   ----  ----                 -------
  Normal  Upgrade  40s   demo.kubebuilder.io  scale replicates
```

> 修改副本数为1：

```bash
[root@master kubebuilder-demo]# kubectl describe Redis/redis-sample
...
Events:
  Type    Reason   Age                    From                 Message
  ----    ------   ----                   ----                 -------
  Normal  Upgrade  4m59s (x2 over 7m39s)  demo.kubebuilder.io  scale replicates
  Normal  Updated  18s (x2 over 65s)      demo.kubebuilder.io  scale replicates
```

### 额外信息的输出

> 默认情况下，我们查看自定义资源Redis时，输出的是这样的：

```bash
[root@master kubebuilder-demo]# kubectl get Redis 
NAME           AGE
redis-sample   21s
```

> 如果我们想要输出中包含副本的数量，即：

```bash
[root@master kubebuilder-demo]# kubectl get Redis
NAME           REPLICAS
redis-sample   5
```

> 关于Addition Printer Columns
>
> ### Additional Printer Columns
>
> > Starting with Kubernetes 1.11, `kubectl get` can ask the server what columns to display. For CRDs, this can be used to provide useful, type-specific information with `kubectl get`, similar to the information provided for built-in types.
>
> The information that gets displayed can be controlled with the additionalPrinterColumns field on your CRD, which is controlled by the
>
```text
+kubebuilder:printcolumn
```
>
> marker on the Go type for your CRD.
>
> > For instance, in the following example, we add fields to display information about the knights, rank, and alias fields from the validation example:
>
 ```go
 // +kubebuilder:printcolumn:name="Alias",type=string,JSONPath=`.spec.alias`
 // +kubebuilder:printcolumn:name="Rank",type=integer,JSONPath=`.spec.rank`
 // +kubebuilder:printcolumn:name="Bravely Run Away",type=boolean,JSONPath=`.spec.knights[?(@ == "Sir Robin")]`,description="when danger rears its ugly head, he bravely turned his tail and fled",priority=10
 // +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
 type Toy struct {
    metavTypeMeta   `json:",inline"`
    metavObjectMeta `json:"metadata,omitempty"`
 
    Spec   ToySpec   `json:"spec,omitempty"`
    Status ToyStatus `json:"status,omitempty"`
 }
 ```

> 这里需要使用到+kubebuilder:printcolumn，关于它的字段描述：
>
> <https://book.kubebuilder.io/reference/markers/crd.html>
>
> ![img](https://img2022.cnblogs.com/blog/1722983/202202/1722983-20220215211950286-911050233.png)
>
> `（1）修改api/v1/redis_types.go`

```go
type RedisStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
	Replicas int `json:"replicas,omitempty"` //添加副本显示
}

//+kubebuilder:object:root=true
//+kubebuilder:subresource:status
//+kubebuilder:printcolumn:JSONPath=".status.replicas",name=Replicas,type=integer

// Redis is the Schema for the redis API
type Redis struct {
	metavTypeMeta   `json:",inline"`
	metavObjectMeta `json:"metadata,omitempty"`

	Spec   RedisSpec   `json:"spec,omitempty"`
	Status RedisStatus `json:"status,omitempty"`
}
```

> `（2）修改controllers/redis_controller.go`

```go
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	...
	//更新Kind Redis状态
	if updateFlag {
         ...
		//更新status状态值
		redis.Status.Replicas = len(redis.Finalizers)
		err := r.Status().Update(ctx, redis)
		if err != nil {
			return ctrl.Result{}, err
		}
	}
	return ctrl.Result{}, nil
}


func (r *RedisReconciler) clearRedis(ctx context.Context, redis *myappvRedis) error {

	...
	//更新status状态值
	redis.Status.Replicas = len(redis.Finalizers)
	err = r.Status().Update(ctx, redis)
	if err != nil {
		return err
	}
	return nil
}
```

> `（3）验证`
>
> make install 后make run

```bash
[root@master kubebuilder-demo]# kubectl get Redis
NAME           REPLICAS
redis-sample   5
[root@master kubebuilder-demo]# kubectl describe Redis/redis-sample
Name:         redis-sample
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  myapp.demo.kubebuilder.io/v1
Kind:         Redis
...
Spec:
  Num:   5
  Port:  6379
Status:
  Replicas:  5
Events:
  Type    Reason   Age   From                 Message
  ----    ------   ----  ----                 -------
  Normal  Upgrade  28s   demo.kubebuilder.io  scale replicates
```
