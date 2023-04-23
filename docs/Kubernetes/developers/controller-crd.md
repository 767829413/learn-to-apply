# 深入理解声明式API-实践自定义控制器

## 利用kuberbuilder快速实践CRD与自定义控制器

### 前提条件

* [kubebuilder安装](https://book.kubebuilder.io/quick-start.html)
* [本地WSL环境搭建](../../docs/Productivity/wsl2-dev.md)
* [安装kubectl](https://kubernetes.io/docs/tasks/tools/)
* [安装kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
* [Kubectl 命令表](http://docs.kubernetes.org.cn/683.html)
* [kubernetes二次开发](../../docs/Kubernetes/kubernetes-secondary-development.md)

我本地的 `golang` `docker` `kubernetes` 版本

```bash
go version
# go version go1.18.2 linux/amd64
```

```bash
kubectl version --output=yaml
```

```yaml
clientVersion:
  buildDate: "2022-05-03T13:46:05Z"
  compiler: gc
  gitCommit: 4ce5a8954017644c5420bae81d72b09b735c21f0
  gitTreeState: clean
  gitVersion: v1.24.0
  goVersion: go1.18.1
  major: "1"
  minor: "24"
  platform: linux/amd64
kustomizeVersion: v4.5.4
serverVersion:
  buildDate: "2022-05-19T15:39:43Z"
  compiler: gc
  gitCommit: 4ce5a8954017644c5420bae81d72b09b735c21f0
  gitTreeState: clean
  gitVersion: v1.24.0
  goVersion: go1.18.1
  major: "1"
  minor: "24"
  platform: linux/amd64
```

```bash
docker version
```

```text
Client: Docker Engine - Community
 Cloud integration: v1.0.25
 Version:           20.10.16
 API version:       1.41
 Go version:        go1.17.10
 Git commit:        aa7e414
 Built:             Thu May 12 09:17:39 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Desktop
 Engine:
  Version:          20.10.16
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.17.10
  Git commit:       f756502
  Built:            Thu May 12 09:15:42 2022
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.4
  GitCommit:        212e8b6fa2f44b9c21b2798135fc6fb7c53efc16
 runc:
  Version:          1.1.1
  GitCommit:        v1.1.1-0-g52de29d
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```

### 步骤一 kubebuilder 初始化操作以及相应Api创建

在自己的GOPATH下面创建一个相面文件夹,相关操作查阅[kubebuilder快速开始文档](https://book.kubebuilder.io/quick-start.html)

```bash
# 项目文件夹创建
mkdir -p $GOPATH/src/kubebuilder-demo
cd $GOPATH/src/kubebuilder-demo

# 使用kubebuilder初始化,等待完成
kubebuilder init --domain demo.kubebuilder.io

# 创建API,根据提示操作
kubebuilder create api --group myapp --version v1 --kind Redis
```

查看下当前项目层级

```bash
tree -L 1
```

```text
.
├── api
├── bin
├── config
├── controllers
├── Dockerfile
├── go.mod
├── go.sum
├── hack
├── main.go
├── Makefile
├── PROJECT
└── README.md

5 directories, 7 files
```

### 步骤二 定义demo开发目标

主要是为了创建自定义 [CRD](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 实现自定义的 `Redis` 类型资源相关 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) 的功能:

* 定义 `CRD` 规格
* 创建 `CRD` 对应的资源 `Redis`
* 实现资源 `Redis` 创建相应 `Pod`,支持 `副本数,验证,扩容收缩`
* 添加对应事件的接收和信息展示

### 步骤三 code

[项目地址](https://github.com/767829413/kubebuilder-demo)

#### 自定义资源字段

```go
## redis_types.go 添加自定义字段
type RedisSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// +kubebuilder:validation:Maximum:=6380
	// +kubebuilder:validation:Minimum:=6370
	Port int `json:"port,omitempty"`

	Num int `json:"num,omitempty"`
}
```

修改测试的自定义资源yaml文件 项目/config/samples/myapp_v1_redis.yaml

```yaml
apiVersion: myapp.demo.kubebuilder.io/v1
kind: Redis
metadata:
  name: redis-sample
spec:
  port: 6379
  num: 4
```

部署自定义资源测试

```bash
kubectl apply -f  config/samples/myapp_v1_redis.yaml
# redis.myapp.demo.kubebuilder.io/redis-sample created
kubectl get Redis
# NAME           AGE
# redis-sample   6s
```

#### 编写控制器业务逻辑

1. **Pod相关操作**

在 `项目/controllers/` 目录下创建 `redis_op.go` 这里主要放一下业务逻辑相关Pod操作

```go
package controllers

import (
	"context"
	"fmt"

	v1 "github.com/767829413/kubebuilder-demo/api/v1"

	coreV1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
)

// Finalizers去重
func GetUniqueFinalizersMap(finalizers []string) map[string]int {
	m := make(map[string]int, len(finalizers))
	for _, v := range finalizers {
		m[v] = 1
	}
	return m
}

// 通过实际请求状态来判断Pod是否存在
func IsRedisPodExist(podName string, redis *v1.Redis, client client.Client) bool {
	err := client.Get(context.Background(), types.NamespacedName{
		Name:      podName,
		Namespace: redis.Namespace,
	}, &coreV1.Pod{})
	return err == nil
}

// 创建redis Pod
func CreateRedisPod(podName string, client client.Client, redisConfig *v1.Redis, scheme *runtime.Scheme) error {
	newPod := &coreV1.Pod{}
	newPod.Name = podName
	newPod.Namespace = redisConfig.Namespace
	newPod.Spec.Containers = []coreV1.Container{
		{
			Name:            redisConfig.Name,
			Image:           "redis:5-alpine",
			ImagePullPolicy: coreV1.PullIfNotPresent,
			Ports: []coreV1.ContainerPort{
				{
					ContainerPort: int32(redisConfig.Spec.Port),
				},
			},
		},
	}
	err := controllerutil.SetControllerReference(redisConfig, newPod, scheme)
	if err != nil {
		return err
	}
	return client.Create(context.Background(), newPod)
}

// 获取自定义多副本的redis pod 名称
func GetRedisPodNames(redis *v1.Redis) []string {
	podNames := make([]string, redis.Spec.Num)
	for i := 0; i < redis.Spec.Num; i++ {
		podNames[i] = fmt.Sprintf("%s-%d", redis.Name, i)
	}
	return podNames
}
```

2. **控制器业务逻辑**

在 `项目/controllers/redis_controller.go` 添加自己的业务流程

```go
/*
Copyright 2022.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controllers

import (
	"context"
	"fmt"

	v1 "github.com/767829413/kubebuilder-demo/api/v1"
	coreV1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/tools/record"
	"k8s.io/client-go/util/workqueue"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/event"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	"sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
	"sigs.k8s.io/controller-runtime/pkg/source"
)

// RedisReconciler reconciles a Redis object
type RedisReconciler struct {
	client.Client
	Scheme        *runtime.Scheme
	EventRecorder record.EventRecorder // 增加事件记录
}

//+kubebuilder:rbac:groups=myapp.demo.kubebuilder.io,resources=redis,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=myapp.demo.kubebuilder.io,resources=redis/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=myapp.demo.kubebuilder.io,resources=redis/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the Redis object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.12.1/pkg/reconcile
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)
	redis := &v1.Redis{}
	// 获取自定义的Redis资源
	err := r.Get(ctx, req.NamespacedName, redis)
	if err != nil {
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}
	// 获取自定义redis资源需要创建的pod名称
	podNames := GetRedisPodNames(redis)
	// 设置一个是否更新的判断
	upFlag := false

	// 删除自定义Redis资源的过程中,需要判断DeletionTimestamp(自动添加)
	// 如果有值表示删除,无值表示未删除
	// 如果缩容时候需要删除相应pod数目
	// len(redis.Finalizers) > redis.Spec.Num 考虑刚开始创建时候 len(redis.Finalizers)必定为0
	// 后续循环的时候,只要没有扩容或者缩容,必定 len(redis.Finalizers) == redis.Spec.Num
	if !redis.DeletionTimestamp.IsZero() || len(redis.Finalizers) > redis.Spec.Num {
		return ctrl.Result{}, r.clearRedis(ctx, redis)
	}
	// 防止重复pod进入Finalizers
	m := GetUniqueFinalizersMap(redis.Finalizers)
	// 创建自定义资源对应的pod
	for _, podName := range podNames {
		if IsRedisPodExist(podName, redis, r.Client) {
			continue
		}
		err := CreateRedisPod(podName, r.Client, redis, r.Scheme)
		if err != nil {
			fmt.Println("create pod failue,", err)
			return ctrl.Result{}, err
		}
		// 如果创建的pod不在finalizers中，则添加
		if _, ok := m[podName]; !ok {
			redis.Finalizers = append(redis.Finalizers, podName)
			upFlag = true
		}
	}

	// 更细自定义资源 Redis 状态
	if upFlag {
		r.EventRecorder.Event(redis, coreV1.EventTypeNormal, "Upgrade", "Capacity expansion and reduction")
		//更新status状态值
		redis.Status.Replicas = len(redis.Finalizers)
		err := r.Status().Update(ctx, redis)
		if err != nil {
			return ctrl.Result{}, err
		}
		err = r.Client.Update(ctx, redis)
		if err != nil {
			return ctrl.Result{}, err
		}
	}

	return ctrl.Result{}, nil
}

// 删除自定义Redis资源和其创建的Pod逻辑
func (r *RedisReconciler) clearRedis(ctx context.Context, redis *v1.Redis) error {
	// 副本数与当前Pod数量的差值就是需要删除的数量
	// 如果相等,删除全部
	// 1. len(redis.Finalizers) > redis.Spec.Num 删除差值数量Pod
	// 2. len(redis.Finalizers) == redis.Spec.Num 删除全部数量Pod
	// 3. len(redis.Finalizers) < redis.Spec.Num 错误的数量,不建议操作
	// 删除后项
	var deletedPodNames []string
	diffNum := len(redis.Finalizers) - redis.Spec.Num
	if diffNum == 0 {
		deletedPodNames = make([]string, len(redis.Finalizers))
		copy(deletedPodNames, redis.Finalizers)
		// 只有设置 Finalizers 为空才能真正删除自定义资源Redis
		redis.Finalizers = []string{}
	} else if diffNum > 0 {
		deletedPodNames = redis.Finalizers[redis.Spec.Num:]
		redis.Finalizers = redis.Finalizers[:redis.Spec.Num]
	}

	for _, finalizer := range deletedPodNames {
		// 通过finalizer来批量删除自定义Redis资源创建的Pod
		err := r.Client.Delete(ctx, &coreV1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      finalizer,
				Namespace: redis.Namespace,
			},
		})
		err = client.IgnoreNotFound(err)
		if err != nil {
			return err
		}
	}
	err := r.Client.Update(ctx, redis)
	if err != nil {
		return err
	}
	err = r.Status().Update(ctx, redis)
	return err
}

// Pod删除时的回调
func (r *RedisReconciler) podDelHandler(event event.DeleteEvent, limitingInterface workqueue.RateLimitingInterface) {
	fmt.Println("deleted pod name is :", event.Object.GetName())
	// 取得OwnerReference,如果自定义资源Kind和Name匹配,则触发reconcile.Request，并加入到等待队列
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
		For(&v1.Redis{}).
		Watches(&source.Kind{
			Type: &coreV1.Pod{},
		}, handler.Funcs{
			DeleteFunc: r.podDelHandler,
		}).
		Complete(r)
}
```

### 注意事项

1. Kubebuilder中的验证是通过 `标签` 的方式来完成

    ```go
    // +kubebuilder:validation:Maximum:=6380
    // +kubebuilder:validation:Minimum:=6370
    Port int `json:"port,omitempty"`
    ```

    [验证规则文档](https://book.kubebuilder.io/reference/markers/crd-validation.html)

2. 针对 `项目/api/v1/xxx_types.go` 的自定义资源字段的修改,都需要执行 `make install` 来生效

    ```bash
    ## 修改后执行
    make install
    ## 查看CRD中关于自定义字段
    kubectl get crds
    # NAME                              CREATED AT
    # redis.myapp.demo.kubebuilder.io       2022-06-30T07:02:33Z 
    kubectl get crds/redis.myapp.demo.kubebuilder.io -o yaml
    ```

3. `Finalizers` 和 `Owner References` 的使用是用来关联Pod资源与自定义资源,防止创建的Pod孤立存在,关于具体如何使用可以参考[Kubernetes Blog](https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/)

4. 额外信息输出 `Additional Printer Columns` 具体[使用文档](https://book.kubebuilder.io/reference/generating-crd.html#additional-printer-columns)

4. 调试方式

    ```bash
    ## 一个terminal执行自定义控制器
    make run
    ## 一个terminal执行自定义资源
    kubectl apply -f /项目目录/config/samples/myapp_v1_xxxx.  yaml
    ## 两相对比进行验证
    ```
