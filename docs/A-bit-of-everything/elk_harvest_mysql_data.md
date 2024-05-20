---
marp: true
---

# 使用ELK采集数据库数据和展示

本次主要探讨如何采集mysql数据并展示,ELK的部署直接快速使用Docker本地搭建即可

---

## 首先配置 logstash/pipeline/logstash.conf 

---

1. 针对 input 需要配置 mysql 采集参数

```conf
input {
	# 获取教研数量
    jdbc {
      # 数据库连接
      jdbc_connection_string => "jdbc:mysql://host:3306/schooldb_infi"
      # 数据库用户
      jdbc_user => "root"
      # 数据库密码
      jdbc_password => "123312341"
      # 数据库连接依赖库文件
      jdbc_driver_library => "/usr/share/logstash/mysql-connector-j-8.0.31.jar"
      # 设备连接类型
	  jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
      # mysql文件, 也可以直接写SQL语句在此处，如下：
      # statement_filepath => "./config/vue_authentication_sys_user.sql"
      statement => "SELECT * FROM `teaching_activity` ta group by org_id"
```

---

```conf
      # 这里类似crontab(分 时 天 月 年)
      schedule => "*/10 * * * *"
      # 是否清除上次采集
      clean_run => true
      # 是否将 字段(column) 名称转小写
      lowercase_column_names => false
      # 指定类型
	  type => "teach_activity_count"
    }
}

```

---

2. 针对 output 需要配置输出参数

```conf
output {
	if[type] == "teach_activity_count" {
        elasticsearch {
            hosts => "elasticsearch:9200"
			user => "logstash_internal"
			password => "${LOGSTASH_INTERNAL_PASSWORD}"
            index => "teach_activity_count"
            document_id => "%{org_id}"
        }
    }
	stdout {}
}
```

---

## 配置 logstash 访问 elasticsearch 的账户权限 logstash_internal

```json
{
  "cluster": [
    "manage_index_templates",
    "monitor",
    "manage_ilm"
  ],
```

---

```json
  "indices": [
    {
      "names": [
        "logs-generic-default",
        "logstash-*",
        "ecs-logstash-*",
        "teach_activity_count",
      ],
      "privileges": [
        "write",
        "create",
        "create_index",
        "manage",
        "manage_ilm"
      ]
    },
```

---

```json
    {
      "names": [
        "logstash",
        "ecs-logstash"
      ],
      "privileges": [
        "create_doc",
        "create",
        "delete",
        "index",
        "write",
        "all",
        "manage"
      ]
    }
  ]
}
```
