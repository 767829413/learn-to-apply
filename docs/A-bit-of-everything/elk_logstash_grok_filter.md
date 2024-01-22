# elk快速搭建调试 grok filter

## 前置条件

* 已经安装docker
* 基本看下elk的介绍和使用文档

## elk本地搭建准备和执行 

### 项目目录结构

```bash
$ tree
.
|-- .env
|-- docker-compose.yml
|-- elasticsearch
|   |-- Dockerfile
|   `-- config
|       `-- elasticsearch.yml
|-- kibana
|   |-- Dockerfile
|   `-- config
|       `-- kibana.yml
`-- logstash
    |-- Dockerfile
    |-- config
    |   `-- logstash.yml
    `-- pipeline
        `-- logstash.conf
```

### 配置文件和Dockerfile

版本的配置放到了 .env, 目前使用的是elk版本是 6.8.23

```bash
$ cat .env 
ELK_VERSION=6.8.23
```

### elasticsearch 目录

```bash
$ cat ./elasticsearch/config/elasticsearch.yml 
---
## Default Elasticsearch configuration from Elasticsearch base image.
## https://github.com/elastic/elasticsearch/blob/6.8/distribution/docker/src/docker/config/elasticsearch.yml
#
cluster.name: "docker-cluster"
network.host: 0.0.0.0

## X-Pack settings
## see https://www.elastic.co/guide/en/elasticsearch/reference/6.8/setup-xpack.html
#
xpack.license.self_generated.type: trial
xpack.security.enabled: true
xpack.monitoring.collection.enabled: true

$ cat ./elasticsearch/Dockerfile 
ARG ELK_VERSION

# https://www.docker.elastic.co/
FROM docker.elastic.co/elasticsearch/elasticsearch:${ELK_VERSION}

# Add your elasticsearch plugins setup here
# Example: RUN elasticsearch-plugin install analysis-icu
```

### kibana 目录

```bash
$ cat ./kibana/config/kibana.yml 
---
## Default Kibana configuration from Kibana base image.
## https://github.com/elastic/kibana/blob/6.8/src/dev/build/tasks/os_packages/docker_generator/templates/kibana_yml.template.js
#
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
xpack.monitoring.ui.container.elasticsearch.enabled: true

## X-Pack security credentials
#
elasticsearch.username: elastic
elasticsearch.password: changeme

$ cat ./kibana/Dockerfile 
ARG ELK_VERSION

# https://www.docker.elastic.co/
FROM docker.elastic.co/kibana/kibana:${ELK_VERSION}

# Add your kibana plugins setup here
# Example: RUN kibana-plugin install <name|url>
```

### logstash 目录

**logstash/pipeline/logstash.conf 后面就是用操作日志过滤**

```bash
$ cat ./logstash/config/logstash.yml 
---
## Default Logstash configuration from Logstash base image.
## https://github.com/elastic/logstash/blob/6.8/docker/data/logstash/config/logstash-full.yml
#
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch:9200" ]

## X-Pack security credentials
#
xpack.monitoring.enabled: true
xpack.monitoring.elasticsearch.username: elastic
xpack.monitoring.elasticsearch.password: changeme

$ cat ./logstash/pipeline/logstash.conf 
input {
        beats {
                port => 5044
        }

        tcp {
                port => 5000
        }

        file {
        path => "/usr/share/logstash/logs/*.log"
        start_position => "beginning"
  }
}

## Add your filters / logstash plugins configuration here

output {
        elasticsearch {
                hosts => "elasticsearch:9200"
                user => "elastic"
                password => "changeme"
        }
}

$ cat ./logstash/Dockerfile 
ARG ELK_VERSION

# https://www.docker.elastic.co/
FROM docker.elastic.co/logstash/logstash:${ELK_VERSION}

# Add your logstash plugins setup here
# Example: RUN logstash-plugin install logstash-filter-json
```

### 配置docker-compose.yaml文件

```bash
$ cat ./docker-compose.yml 
version: '3.2'

services:
  elasticsearch:
    build:
      context: elasticsearch/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro,z
      - elasticsearch:/usr/share/elasticsearch/data:z
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: "-Xmx256m -Xms256m"
      ELASTIC_PASSWORD: changeme
      # Use single node discovery in order to disable production mode and avoid bootstrap checks.
      # see: https://www.elastic.co/guide/en/elasticsearch/reference/6.8/bootstrap-checks.html
      discovery.type: single-node
    networks:
      - elk

  logstash:
    build:
      context: logstash/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro,z
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro,z
      # 设置目录,方便文件夹下日志抓取
      - D:/elk/logs:/usr/share/logstash/logs
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    build:
      context: kibana/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      - ./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml:ro,z
    ports:
      - "5601:5601"
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch:
```

### 执行

```bash
$ docker-compose up 
time="2024-01-21T10:18:09+08:00" level=warning msg="mount of type `volume` should not define `bind` option"
[+] Running 5/5
 ✔ Network docker-elk_elk                Created                                                                                                                                                          0.0s 
 ✔ Volume "docker-elk_elasticsearch"     Created                                                                                                                                                          0.0s 
 ✔ Container docker-elk-elasticsearch-1  Created                                                                                                                                                          0.1s 
 ✔ Container docker-elk-logstash-1       Created                                                                                                                                                          0.1s 
 ✔ Container docker-elk-kibana-1         Created                                                                                                                                                          0.1s 
Attaching to elasticsearch-1, kibana-1, logstash-1
......
```

## Grok 正则捕获

Grok 是 Logstash 最重要的插件。

可以在 grok 里预定义好命名正则表达式，在稍后(grok参数或者其他正则表达式里)引用它

### 正则表达式语法

在 grok 里写标准的正则，像下面这样：

```text
\s+(?<request_time>\d+(?:\.\d+)?)\s+
```

在 `logstash/pipeline/logstash.conf` 添加第一个过滤器区段配置。

配置要添加在输入和输出区段之间(logstash 执行区段的时候并不依赖于次序，不过为了看得方便，建议按次序书写)：

```conf
input {
        beats {
                port => 5044
        }

        tcp {
                port => 5000
        }

        file {
        path => "/usr/share/logstash/logs/*.log"
        start_position => "beginning"
  }
}

filter {
    grok {
        match => {
            "message" => "\s+(?<request_time>\d+(?:\.\d+)?)\s+"
        }
    }
}

output {
        elasticsearch {
                hosts => "elasticsearch:9200"
                user => "elastic"
                password => "changeme"
        }
}
```

这里可以在 D:/elk/logs 目录下添加一个文件 test.log,向里面输入一段内容

```bash
$ cat D:/elk/logs/test.log
begin 123.456 end
```

然后在 <http://localhost:5601/> 的 kibana 页面就能看到数据了

![data](https://pic.imgdb.cn/item/65add483871b83018accd3b3.jpg)

Grok 支持把预定义的 grok 表达式 写入到文件中，官方提供的预定义 grok 表达式见：

<https://github.com/logstash/logstash/tree/v1.4.2/patterns>

### 实践使用

剩下的就可以根据正则自己定义了,这里举几个例子

```conf
filter {
    grok {
        match => {
            "message" => [
                    # begin 123.456 end
                    "\s+(?<request_time>\d+(?:\.\d+)?)\s+",
					# [18:35:29.177] INFO   TraceId:9b274f080e3b00dc35b92406c4c1c3df,ReqId:liveclass_service1656,deal over and duration:27.982009ms |cn-hangzhou.192.168.202.45|liveclass-service-585fbcb56-pk2j7
					"\[(?<localTime>.*?)\]%{SPACE}%{LOGLEVEL:level}%{SPACE}TraceId:(?<tid>[\w\d]+)\,(?<logger>.*?)\,(?<logData>.*)",

                    # [18:35:33.206] INFO   [POST](/tk/teachingActivity/v2/getDetailForTrWeb)<<<<TraceId:029851b7b257c79705102b3f22e86194,ReqId:liveclass_service1657,response JSON data:{"code":0,"reqId":"liveclass_service1657","reqTime":1705401333191,"obj":{"teachingActivity":{"id":1406,"orgId":78,"speakerUserId":null,"speakerUserName":"-"}}} |cn-hangzhou.192.168.202.45|liveclass-service-585fbcb56-pk2j7
                    "\[(?<localTime>.*?)\]%{SPACE}%{LOGLEVEL:level}%{SPACE}(?<logger>.*?)<<<<TraceId:(?<tid>[\w\d]{32}),(?<logData>.*)",

                    # [09:49:11.947] INFO   [GET](/inner/getDeployments)>>>>TraceId:cbae6ddd9ee32e50421e236a9747a23f,ReqId:balance4school206391,from:172.16.1.35,body:{} |cn-hangzhou.172.16.1.23|balance-65bcf5998c-kr6qn

					# [18:35:33.191] INFO   [POST](/tk/teachingActivity/v2/getDetailForTrWeb)>>>>TraceId:029851b7b257c79705102b3f22e86194,ReqId:liveclass_service1657,from:49.79.30.10,body:{"id":1406} |cn-hangzhou.192.168.202.45|liveclass-service-585fbcb56-pk2j7
					"\[(?<localTime>.*?)\]%{SPACE}%{LOGLEVEL:level}%{SPACE}(?<logger>.*?)>>>>TraceId:(?<tid>[\w\d]{32}),(?<logData>.*)",

					# [18:35:41.209] INFO   TraceId:27c06060713e438f5637f9d870fa99cb [GORM] /src/dao/teachingActivityDao/teachingActivity.dao.go:553 [INFO] [3.169ms] [rows:1] SELECT teaching_activity.*,teaching_activity_group.visitor_join,teaching_activity_group.register_form,teaching_activity_group.cover FROM `teaching_activity` left join teaching_activity_group on teaching_activity_group.id = teaching_activity.teaching_project_id WHERE teaching_activity.id = 1405 AND `teaching_activity`.`deleted_at` IS NULL ORDER BY `teaching_activity`.`id` LIMIT 1 |cn-hangzhou.192.168.202.45|liveclass-service-585fbcb56-pk2j7
					"\[(?<localTime>.*?)\]%{SPACE}%{LOGLEVEL:level}%{SPACE}TraceId:(?<tid>[\w\d]{32})%{SPACE}\[(?<logger>.*?)\]%{SPACE}(?<logData>.*)",

					# [18:35:41.205] INFO   {"code":0,"component":"grpc","kind":"client","latency":0.004662558,"operation":"/api.tokenManage.TokenCheck/GetInternalToken","params":{"token":"Visitor_65a5fc4aedcd2b0001ed5555_788bd0ba-aac2-4429-92e6-eceb71eb498b","userType":7},"reason":"","response":{"visitorUserInfo":{"meetingId":"bc53e111-6b39-4acc-bd99-fc21cfb775f2","userId":"65a5fc4aedcd2b0001ed5555","userType":7,"userName":"张义峰"}},"span.id":"1c99974c7e30f5ad","stack":"","statusCode":0,"trace.id":"27c06060713e438f5637f9d870fa99cb"} |cn-hangzhou.192.168.202.45|liveclass-service-585fbcb56-pk2j7
					"\[(?<localTime>.*?)\]%{SPACE}%{LOGLEVEL:level}%{SPACE}(?<logData>.*)\"trace\.id\":\"(?<tid>[\w\d]{32})\"\}(?<logger>.*?)",

                    # [13:34:41.420] INFO   [GORM] C:/plasoWorkSpace/infi_sdk_service/dao/userDao/user.dao.go:37 [INFO] [86.046ms] [rows:1] SELECT * FROM `tbl_user` WHERE (phone = '17601542658' and status = 1 ) AND `tbl_user`.`deleted_at` IS NULL ORDER BY `tbl_user`.`id` LIMIT 1  @TraceId:a01d77cf16c821a456c769b010c68465
                    "\[(?<localTime>.*?)\]%{SPACE}%{LOGLEVEL:level}%{SPACE}\[GORM\]%{SPACE}(?<logger>.*?)%{SPACE}\[INFO\](?<logData>.*)%{SPACE}\@TraceId\:(?<tid>[\w\d]{32})",

                    # {"key":"name","name":"姓名","fieldType":"TEXT","required":2,"editable":false,"value":null},{"key":"school","name":"学校","fieldType":"TEXT","required":1,"editable":true,"value":null}],"beginTime":1704261600000,"endTime":1704279600000,"dispBeginTime":1704261600000,"dispEndTime":1704279600000,"createUser":255,"canShare":1,"createdAt":1704259557463,"updatedAt":1704274296706,"deletedAt":null,"status":3,"createUserName":"蔡志东"}]},"traceId":"d5099972d4e2ea227618110b1228a48b"} |cn-hangzhou.192.168.202.45|liveclass-service-585fbcb56-pk2j7
                    "(?<logData>.*)\"traceId\":\"(?<tid>[\w\d]{32})\"\}(?<logger>.*?)",


                    # 其他未匹配
                    "(?<logData>.*)"
                ]
        }
    }
}
```

大概的效果就是: 

![disp](https://pic.imgdb.cn/item/65add89b871b83018ad76e92.jpg)