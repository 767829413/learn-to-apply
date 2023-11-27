# rabbitmq Virtual Hosts 使用

## 概念

[官方文档介绍](https://www.rabbitmq.com/vhosts.html)

```text
RabbitMQ 是一个多租户系统：连接、交换、队列、绑定、用户权限、策略和其他一些东西都属于虚拟主机，即实体的逻辑组。如果您熟悉 Apache 中的虚拟主机或 Nginx 中的服务器块，那么它们的理念是相似的。

但有一个重要区别：Apache 中的虚拟主机是在配置文件中定义的；而 RabbitMQ 则不是：虚拟主机是使用 rabbitmqctl 或 HTTP API 创建和删除的。
```

## 使用流程

1. 本地docker搭建一个 rabbitmq

```bash
docker run -d -p 5672:5672 -p 15672:15672 --hostname my-rabbit --name rabbit -e RABBITMQ_DEFAULT_USER=user -e RABBITMQ_DEFAULT_PASS=user rabbitmq:3-management
```

2. 创建虚拟主机

```bash
curl -u user:user -X PUT http://172.93.221.226:15672/api/vhosts/vhost1
curl -u user:user -X PUT http://172.93.221.226:15672/api/vhosts/vhost2
curl -u user:user -X PUT http://172.93.221.226:15672/api/vhosts/vhost3
curl -u user:user -X PUT http://172.93.221.226:15672/api/vhosts/vhost4
```

3. 通过代码验证一下

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/streadway/amqp"
)

var (
	user = "user"
	pwd  = "user"
	addr = "mw.internal.cn"
	port = "5672"

	url = fmt.Sprintf(
		"amqp://%s:%s@%s:%s/",
		user,
		pwd,
		addr,
		port,
	)

	config = amqp.Config{
		Vhost:      "/", //设置服务的Vhost
		Properties: amqp.Table{},
	}
	config1 = amqp.Config{
		Vhost:      "vhost1",
		Properties: amqp.Table{},
	}
	config2 = amqp.Config{
		Vhost:      "vhost2",
		Properties: amqp.Table{},
	}
	config3 = amqp.Config{
		Vhost:      "vhost3",
		Properties: amqp.Table{},
	}
	config4 = amqp.Config{
		Vhost:      "vhost4",
		Properties: amqp.Table{},
	}
)

func failOnError(err error, msg string) {
	if err != nil {
		log.Fatalf("%s: %s", msg, err)
	}
}

func main() {
	c, cancel := context.WithCancel(context.Background())
	go exec(c, url, config)
	go exec(c, url, config1)
	go exec(c, url, config2)
	go exec(c, url, config3)
	go exec(c, url, config4)

	time.Sleep(time.Second * 40)
	cancel()
}

func exec(c context.Context, url string, config amqp.Config) {
	conn, err := amqp.DialConfig(url, config)
	failOnError(err, config.Vhost+": Failed to connect to RabbitMQ")
	defer conn.Close()
	// 创建一个通道
	ch, err := conn.Channel()
	failOnError(err, config.Vhost+": Failed to open a channel")
	defer ch.Close()

	// 声明一个队列
	q, err := ch.QueueDeclare(
		config.Vhost+": hello", // 队列名称
		false,                  // 是否持久化
		false,                  // 是否自动删除
		false,                  // 是否独占模式
		false,                  // 是否阻塞
		nil,                    // 额外属性
	)
	failOnError(err, config.Vhost+": Failed to declare a queue")
	var ok = make(chan struct{})
	// 发送消息到队列
	go func() {
		var seq = 1
		for {
			body := fmt.Sprintf(config.Vhost+": Hello, RabbitMQ---%d!", seq)
			err = ch.Publish(
				"",     // 交换机名称
				q.Name, // 队列名称
				false,  // 是否强制
				false,  // 是否立即发送
				amqp.Publishing{
					ContentType: "text/plain",
					Body:        []byte(body),
				})
			failOnError(err, config.Vhost+": Failed to publish a message")
			fmt.Println(config.Vhost + ": Message sent!")
			ok <- struct{}{}
			seq++
			time.Sleep(2 * time.Second)
		}
	}()

	// 接收来自队列的消息
	msgs, err := ch.Consume(
		q.Name, // 队列名称
		"",     // 消费者名称
		true,   // 是否自动应答
		false,  // 是否独占模式
		false,  // 是否阻塞
		false,  // 是否等待
		nil,    // 额外属性
	)
	failOnError(err, config.Vhost+": Failed to register a consumer")

	// 处理接收到的消息
	go func() {
		for msg := range msgs {
			fmt.Println(config.Vhost+": Received message:", string(msg.Body))
			<-ok
		}
	}()

	// 等待程序退出
	<-c.Done()
}
```

4. 通过代码验证一下

```bash
go run main.go
/: Message sent!
vhost1: Message sent!
vhost3: Message sent!
vhost2: Message sent!
vhost4: Message sent!
/: Received message: /: Hello, RabbitMQ---1!
vhost1: Received message: vhost1: Hello, RabbitMQ---1!
vhost3: Received message: vhost3: Hello, RabbitMQ---1!
vhost2: Received message: vhost2: Hello, RabbitMQ---1!
vhost4: Received message: vhost4: Hello, RabbitMQ---1!
......
```