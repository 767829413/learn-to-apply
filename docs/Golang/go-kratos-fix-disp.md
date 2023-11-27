# 简单实现go-kratos固定本地服务发现

[Go GRPC如何接入服务发现，解析go-kratos的etcd注册流程](https://www.jianshu.com/p/d89db454720d)

首先看了一下网上相关的教学，了解以下go-kratos的etcd注册和发现流程,因为后续只要固定请求的rpc地址即可就做了尝试

[go-kratos的服务注册与发现](https://go-kratos.dev/docs/component/registry)

Registry 接口分为两个，Registrar 为实例注册和反注册，Discovery 为服务实例列表获取,这里只要实现 Discovery就行了

// 首先建议一个简单的rpc服务端和客户端

项目结构：

```bash
.
├── api
│   └── hello
│       ├── hello.pb.go
│       └── hello.proto
├── client
│   └── client.go
├── go.mod
├── go.sum
├── main.go
└── server
    └── server.go
```

1. 编写描述文件：hello.proto

```bash
cd ./api/hello
cat >> ./hello.proto << EOF
syntax = "proto3"; // 指定proto版本
package hello;     // 指定默认包名

// 指定golang包名
option go_package = "hello";

// 定义Hello服务
service Hello {
    // 定义SayHello方法
    rpc SayHello(HelloRequest) returns (HelloResponse) {}
}

// HelloRequest 请求结构
message HelloRequest {
    string name = 1;
}

// HelloResponse 响应结构
message HelloResponse {
    string message = 1;
}
EOF
```

hello.proto文件中定义了一个Hello Service，该服务包含一个SayHello方法，同时声明了HelloRequest和HelloResponse消息结构用于请求和响应。客户端使用HelloRequest参数调用SayHello方法请求服务端，服务端响应HelloResponse消息

2. 编译生成.pb.go文件

```bash
# 编译hello.proto
$ protoc -I . --go_out=plugins=grpc:. ./hello.proto
```

在当前目录内生成的hello.pb.go文件，按照.proto文件中的说明，包含服务端接口HelloServer描述，客户端接口及实现HelloClient，及HelloRequest、HelloResponse结构体

3. 实现server接口

```go
package main

import (
	"context"
	"fmt"
	"github.com/go-kratos/kratos/v2/middleware/recovery"
	"github.com/go-kratos/kratos/v2/transport/grpc"
	"time"

	"github.com/767829413/tmp-exec/api/hello"
	"github.com/go-kratos/kratos/v2"
	"github.com/go-kratos/kratos/v2/middleware/tracing"
	"os"
)

var (
	// Name is the name of the compiled software.
	Name string = "liveclass"
	// Version is the version of the compiled software.
	Version = "1.0.0"
	id, _   = os.Hostname()
	address = fmt.Sprintf("0.0.0.0:%d", 8888)
)

func main() {
	app := kratos.New(
		kratos.ID(id),
		kratos.Name(Name),
		kratos.Version(Version),
		kratos.Metadata(map[string]string{}),
		kratos.Server(
			NewGRPCServer(),
		),
	)
	// start and wait for stop signal
	if err := app.Run(); err != nil {
		panic(err)
	}
}

func NewGRPCServer() *grpc.Server {
	var opts = []grpc.ServerOption{
		grpc.Middleware(
			recovery.Recovery(),
			tracing.Server(),
		),
		grpc.Timeout(time.Second * 60),
	}
	opts = append(opts, grpc.Address(address))
	srv := grpc.NewServer(opts...)
	hello.RegisterHServer(srv, HelloService)
	return srv
}

// 定义helloService并实现约定的接口
type helloService struct{}

// HelloService Hello服务
var HelloService = helloService{}

// SayHello 实现Hello服务接口
func (h helloService) SayHello(ctx context.Context, in *hello.HelloRequest) (*hello.HelloResponse, error) {
	resp := new(hello.HelloResponse)
	resp.Message = fmt.Sprintf("This is ?????? Hello %s.", in.Name)

	return resp, nil
}
```

4. 实现client接口

```go
package main

import (
	"context"
	"fmt"
	"github.com/767829413/tmp-exec/grpc/api/hello"
	"github.com/go-kratos/kratos/v2/log"
	"github.com/go-kratos/kratos/v2/middleware/recovery"
	"github.com/go-kratos/kratos/v2/middleware/tracing"
	reg "github.com/go-kratos/kratos/v2/registry"
	"github.com/go-kratos/kratos/v2/transport/grpc"
	grpc2 "google.golang.org/grpc"
	"time"
)

var (
	grpcConn1 *grpc2.ClientConn

	livecSrvAddr = fmt.Sprintf("127.0.0.1:%d", 8888)
	livecSrv     = &reg.ServiceInstance{
		ID:        "liveclass_service",
		Name:      "liveclass_service",
		Version:   "1.0.0",
		Metadata:  map[string]string{},
		Endpoints: []string{livecSrvAddr},
	}


)

func main() {
	InitInternalRpcClientNew()
	conn := GetHelloConnection()
	resp, err := conn.SayHello(context.Background(), &hello.HelloRequest{Name: "kratos"})
	if err != nil {
		log.Error("error: ", err)
		return
	}
	log.Info(resp.Message)
}

func InitInternalRpcClientNew() {
	var err error
	dis := new(livecSrv)
	//连接grpc服务
	ctx1, cel := context.WithTimeout(context.Background(), time.Second*3600)
	defer cel()
	grpcConn1, err = grpc.DialInsecure(
		ctx1,
		grpc.WithEndpoint(livecSrvAddr),
		grpc.WithMiddleware(
			recovery.Recovery(),
			tracing.Client(),
		),
		grpc.WithTimeout(time.Second*3600),
		grpc.WithDiscovery(dis),
	)
	if err != nil {
		log.Fatal(err)
	}
}

func GetConnectionNew() *grpc2.ClientConn {
	return grpcConn1
}

func GetHelloConnection() hello.HelloClient {
	return hello.NewHelloClient(GetConnectionNew())
}

type fixedDiscovery struct {
	fixSer *reg.ServiceInstance
}

func new(ins *reg.ServiceInstance) *fixedDiscovery {
	return &fixedDiscovery{fixSer: ins}
}

// 根据 serviceName 直接拉取实例列表
func (fdis *fixedDiscovery) GetService(ctx context.Context, serviceName string) ([]*reg.ServiceInstance, error) {
	return []*reg.ServiceInstance{fdis.fixSer}, nil
}

// 根据 serviceName 阻塞式订阅一个服务的实例列表信息
func (fdis *fixedDiscovery) Watch(ctx context.Context, serviceName string) (reg.Watcher, error) {
	w := &watcher{fixSer: fdis.fixSer}
	return w, nil
}

var _ reg.Watcher = (*watcher)(nil)

type watcher struct {
	fixSer *reg.ServiceInstance
}

func (w *watcher) Next() ([]*reg.ServiceInstance, error) {
	return []*reg.ServiceInstance{w.fixSer}, nil
}

func (w *watcher) Stop() error {
	return nil
}
```

5. 这里需要修改一下 hello.pb.go 做个兼容

```go
// hello.pb.go 添加了RegisterHServer方法做兼容
func RegisterHServer(s grpc.ServiceRegistrar, srv HelloServer) {
	s.RegisterService(&_Hello_serviceDesc, srv)
}
```

6. 进行测试验证以下

server

```bash
$ go run ./server/server.go 
INFO msg=[gRPC] server listening on: [::]:8888

```

client

```bash
$ go run ./client/client.go 
INFO msg=This is ?????? Hello kratos.
```
