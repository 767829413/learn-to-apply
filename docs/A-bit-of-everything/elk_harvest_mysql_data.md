---
marp: true
---

# 使用ELK采集数据库数据和展示

本次主要探讨如何采集mysql数据并展示,ELK的部署直接快速使用Docker本地搭建即可

---

## 首先配置 logstash/pipeline/logstash.conf 

1. 针对 input 需要配置 mysql 采集参数

```conf
input {
	# 获取教研数量
    jdbc {
      # 数据库连接
      jdbc_connection_string => "jdbc:mysql://mysql.rongke-base:3306/schooldb_infi?charset=utf8mb4&parseTime=true&loc=Local"
      # 数据库用户
      jdbc_user => "root"
      # 数据库密码
      jdbc_password => "04030201"
      # 数据库连接依赖库文件
      jdbc_driver_library => "/usr/share/logstash/mysql-connector-j-8.0.31.jar"
      # 设备连接类型
	  jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
      # mysql文件, 也可以直接写SQL语句在此处，如下：
      # statement_filepath => "./config/vue_authentication_sys_user.sql"
      statement => "SELECT org_id,COUNT(*) teach_activity_count ,SUM(ta.duration) teach_activity_duration,SUM(ta.max_online_num) max_online_num ,SUM(ta.accumulate_online_num) accumulate_online_num ,SUM(ta.register_num) register_num FROM `teaching_activity` ta group by org_id"
      
      # 这里类似crontab,可以定制定时操作，比如每分钟执行一次同步(分 时 天 月 年)
      schedule => "*/10 * * * *"
      # 是否清除 last_run_metadata_path 的记录,如果为真那么每次都相当于从头开始查询所有的数据库记录
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
