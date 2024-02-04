# 使用goose进行SQL文件迁移

## 介绍

1. 数据库模式迁移是对关系数据库模式进行增量、可逆更改和版本控制的管理

2. 迁移基本上是跟踪数据库模式的细粒度变更，这些变更会以单独的脚本文件形式反映出来

3. [goose](https://github.com/pressly/goose)是一款数据库迁移工具, 它允许你通过创建增量 SQL 变更或 Go 函数来管理数据库模式,这里介绍SQL模式

4. SQL迁移文件相关内容参考: <https://pressly.github.io/goose/blog/2022/overview-sql-file/> 

5. 命令执行参考: <https://pressly.github.io/goose/blog/2021/visualizing-up-down-commands/#down-examples>

## 安装

使用 golang 进行本地安装, 在具有 golang 环境机器是运行

```bash
go install github.com/pressly/goose/v3/cmd/goose@latest
```

执行命令判断是否成功

```bash
goose --version
goose version: v3.17.0
```

## 开始使用

1. 创建迁移文件

```bash
# 这个是使用时间戳作为顺序
goose -dir ./sql create 1_29_add_admin_user_table sql
2024/01/31 15:54:56 Created new file: sql/20240131075456_1_29_add_admin_user_table.sql

# 这个是指定数字作为顺序
goose -dir ./sql -s create add_admin_user_table sql
2024/01/31 16:23:25 Created new file: sql/00001_1_29_add_admin_user_table.sql
```

2. 迁移文件内容

一个 SQL 迁移文件可以同时包含向上和向下迁移

有一个开放问题 [#374](https://github.com/pressly/goose/issues/374) 要求支持将迁移分拆成不同的文件。

每个 SQL 迁移文件都应包含一个 `-- +goose Up` 向上注释。

`-- +goose Down` 向下注释是可选的，但建议使用，而且必须在文件中的向上注释之后

下面是一个示例

```sql
-- +goose Up
CREATE TABLE
  `admin_user` (
    `id` int NOT NULL AUTO_INCREMENT,
    `user_name` varchar(255) DEFAULT NULL COMMENT '用户姓名',
    `login_name` varchar(255) NOT NULL COMMENT '登录账号',
    `password` varchar(255) NOT NULL COMMENT '用户密码',
    `created_at` bigint NOT NULL COMMENT '创建时间',
    `updated_at` bigint NOT NULL COMMENT '更新时间',
    `deleted_at` bigint DEFAULT NULL COMMENT '删除时间',
    PRIMARY KEY (`id`, `login_name`) USING BTREE
  ) ENGINE = InnoDB AUTO_INCREMENT = 0;

INSERT INTO `schooldb`.`admin_user` (`id`, `user_name`, `login_name`, `password`, `created_at`, `updated_at`, `deleted_at`)
    VALUES (1, 'admin', 'plasoadmin', '9fc41178fc6777db7be124e2b79b3573', 1669714970000, 1669714970000, NULL);

-- +goose Down
DROP TABLE IF EXISTS `admin_user`;
```

在 `-- +goose Up` 之后的任何语句都将作为向上迁移的一部分执行, 而在 `-- +goose Down` 之后的任何语句都将作为向下迁移的一部分执行。

默认情况下，SQL 语句以分号分隔, 实际情况是查询语句必须以分号结束，才能被 goose 正确识别。

如果是复杂的内容,建议使用 `-- +goose StatementBegin` 和 `-- +goose StatementEnd` 进行上下包裹,比如: 

```sql
-- +goose Up

-- +goose StatementBegin
CREATE OR REPLACE FUNCTION histories_partition_creation( DATE, DATE )
returns void AS $$
DECLARE
  create_query text;
BEGIN
-- This comment will be preserved.
  -- And so will this one.
  FOR create_query IN SELECT
      'CREATE TABLE IF NOT EXISTS histories_'
      || TO_CHAR( d, 'YYYY_MM' )
      || ' ( CHECK( created_at >= timestamp '''
      || TO_CHAR( d, 'YYYY-MM-DD 00:00:00' )
      || ''' AND created_at < timestamp '''
      || TO_CHAR( d + INTERVAL '1 month', 'YYYY-MM-DD 00:00:00' )
      || ''' ) ) inherits ( histories );'
    FROM generate_series( $1, $2, '1 month' ) AS d
  LOOP
    EXECUTE create_query;
  END LOOP;  -- LOOP END
END;         -- FUNCTION END
$$
language plpgsql;
-- +goose StatementEnd
```

当 goose 检测到 `-- +goose StatementBegin` 注释时，它会继续解析语句，忽略分号，直到检测到 `-- +goose StatementEnd`, 语句中的注释和空行将被保留！

StatementBegin 和 StatementEnd 注解还可用于合并多条语句，使其作为一条命令发送，而不是逐条发送。

通过下面的示例可以很好地说明这一点: 

假设我们进行了一次迁移，创建了一个用户表，并通过不同的 INSERT 增加了 100,000 条记录。

```sql
-- +goose Up
CREATE TABLE users (
    id int NOT NULL PRIMARY KEY,
    username text,
    name text,
    surname text
);


INSERT INTO "users" ("id", "username", "name", "surname") VALUES (1, 'gallant_almeida7', 'Gallant', 'Almeida7');
INSERT INTO "users" ("id", "username", "name", "surname") VALUES (2, 'brave_spence8', 'Brave', 'Spence8');
.
.
INSERT INTO "users" ("id", "username", "name", "surname") VALUES (99999, 'jovial_chaum1', 'Jovial', 'Chaum1');
INSERT INTO "users" ("id", "username", "name", "surname") VALUES (100000, 'goofy_ptolemy0', 'Goofy', 'Ptolemy0');

-- +goose Down
DROP TABLE users;
```

up 包含 100001 条唯一语句，所有语句都在同一事务中执行，但逐一发送到数据库。由于往返次数较多，完成迁移需要接近 38 秒。

如果将插入合并为一条命令，可以用 `--+goose StatementBegin` 和 `--+goose StatementEnd` 注解将它们封装起来。

需要注意的是，一条命令仍然包含多条以分号分隔的语句，但它们会以相同的查询字符串发送到服务器，就像下面这样"insert into ...；insert into ...；"。

**同时可能由格式化的问题,实际使用中不太建议**

```sql
-- +goose Up
CREATE TABLE users (
    id int NOT NULL PRIMARY KEY,
    username text,
    name text,
    surname text
);

-- +goose StatementBegin
INSERT INTO "users" ("id", "username", "name", "surname") VALUES (1, 'gallant_almeida7', 'Gallant', 'Almeida7');
INSERT INTO "users" ("id", "username", "name", "surname") VALUES (2, 'brave_spence8', 'Brave', 'Spence8');
.
.
INSERT INTO "users" ("id", "username", "name", "surname") VALUES (99999, 'jovial_chaum1', 'Jovial', 'Chaum1');
INSERT INTO "users" ("id", "username", "name", "surname") VALUES (100000, 'goofy_ptolemy0', 'Goofy', 'Ptolemy0');
-- +goose StatementEnd

-- +goose Down
DROP TABLE users;
```

goose 将一次发送一条命令，现在这条命令由多个以分号分隔的语句组成。

是的，这是一个较大的有效载荷，但没关系，迁移将在接近 3s 内执行，与之前运行接近 38s 的示例相比，速度快了一个数量级。

一般迁移文件中的所有语句都在事务中进行的。但是某些语句，比如 CREATE DATABASE 或 CREATE INDEX CONCURRENTLY，不能在事务块中运行。

在这种情况下，请添加 `-- +goose NO TRANSACTION` 注释，通常放在文件顶部。

该注释指示 goose 在文件中运行所有语句，而不带事务。这适用于文件中的所有 up 和 down 语句。

```sql
-- +goose NO TRANSACTION

-- +goose Up
CREATE INDEX CONCURRENTLY ON users (user_id);

-- +goose Down
DROP INDEX IF EXISTS users_user_id_idx;
```

## 执行迁移和回滚

### 导出变量

首先声明要使用的数据库以及与数据库的连接字符串。从 goose 3.2.0 版开始，数据库驱动程序和数据库连接字符串将分别使用 GOOSE_DRIVER 和 GOOSE_DBSTRING 变量。使用这些导出命令来定义,详情可以使用 `goose --help` 命令查看

```bash
# 导出变量
# export GOOSE_DRIVER=mysql
# export GOOSE_DBSTRING="user:password@/dbname"

# 也可以直接通过 option 配置
# goose mysql "user:password@/db_name?charset=utf8mb4&parseTime=true&loc=Local" status
```

### 初始化操作

第一次使用 goose 的时候可以执行

```bash
goose -dir ./sql mysql "user:password@/db_name?charset=utf8mb4&parseTime=true&loc=Local" status
2024/02/01 17:21:20     Applied At                  Migration
2024/02/01 17:21:20     =======================================
2024/02/01 17:21:20     Pending                  -- 00001_1_29_init_table1_sql.sql
2024/02/01 17:21:20     Pending                  -- 00002_1_31_add_table2_sql.sql
2024/02/01 17:21:20     Pending                  -- 00003_1_32_add_table3_sql.sql
2024/02/01 17:21:20     Pending                  -- 00004_1_33_add_table4_sql.sql
2024/02/01 17:21:20     Pending                  -- 00005_1_34_add_table5_sql.sql
2024/02/01 17:21:20     Pending                  -- 00006_1_35_add_table6_sql.sql
2024/02/01 17:21:20     Pending                  -- 00007_1_36_add_table7_sql.sql
```

会在数据创建新表 `goose_db_version` 

```sql
CREATE TABLE
  `goose_db_version` (
    `id` bigint unsigned NOT NULL AUTO_INCREMENT,
    `version_id` bigint NOT NULL,
    `is_applied` tinyint(1) NOT NULL,
    `tstamp` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    UNIQUE KEY `id` (`id`)
  ) ENGINE = InnoDB;
```

获取当前数据版本:

```bash
goose -dir ./sql mysql "user:password@/db_name?charset=utf8mb4&parseTime=true&loc=Local" version
2024/02/01 17:23:55 goose: version 0
```

版本为 0, 表示暂未进行任何一次的迁移

### 命令的使用

#### up, up-by-one, up-to, down, down-to

假设我们的 ./sql 文件夹包含 7 个迁移文件。我们之前已经应用了 4 次迁移，还有 3 次迁移正在等待中。

执行 `goose version` ,此时应该返回 goose: version 4

```bash
goose -dir ./sql mysql "user:password@/db_name?charset=utf8mb4&parseTime=true&loc=Local" version
2024/02/01 17:23:55 goose: version 4
```

执行 `goose status` 此时返回:

```bash
goose -dir ./sql mysql "user:password@/db_name?charset=utf8mb4&parseTime=true&loc=Local" status
2024/02/01 17:21:20     Applied At                  Migration
2024/02/01 17:21:20     =======================================
2024/02/01 17:21:20     Thu Feb  1 17:21:09 2024 -- 00001_1_29_init_table1_sql.sql
2024/02/01 17:21:20     Thu Feb  1 17:21:09 2024 -- 00002_1_29_add_table2_sql.sql
2024/02/01 17:21:20     Thu Feb  1 17:21:09 2024 -- 00003_1_29_add_table3_sql.sql
2024/02/01 17:21:20     Thu Feb  1 17:21:09 2024 -- 00004_1_29_add_table4_sql.sql
2024/02/01 17:21:20     Pending                  -- 00005_1_29_add_table5_sql.sql
2024/02/01 17:21:20     Pending                  -- 00006_1_29_add_table6_sql.sql
2024/02/01 17:21:20     Pending                  -- 00007_1_29_add_table7_sql.sql
```

1. UP 操作

    * 执行 `goose up` 应用所有 3 个待定迁移：5 6 和 7,表示执行所有迁移 `Pending` 状态的迁移版本
    * 执行 `goose up-by-one` 仅仅只迁移 5 的任务
    * 执行 `goose up-to 4` 没有任何作用，因为 4 已被显示迁移完成状态 `Thu Feb  1 17:21:09 2024`
    * 执行 `goose up-to 6` 应用迁移 5 和 6

    ![up](https://pic.imgdb.cn/item/65bc3f65871b83018adad764.jpg)

2. DOWN 操作

    * 执行 `goose down` 应用回滚迁移到版本 3
    * 执行 `goose down-to 4` 没有任何作用
    * 执行 `goose down-to 2` 应用执行回滚 4 3
    * 执行 `goose down-to 0` 全部回滚

    ![down](https://pic.imgdb.cn/item/65bc3f65871b83018adad6fa.png)

### 已有项目添加迭代方案

1. 执行 `status` 命令查看迁移状态

```bash
goose -dir ./sql/migrations/ mysql "user:passwd@tcp(mysql.rongke-base:3306)/your_db?charset=utf8mb4&parseTime=true&loc=Local" status
2024/02/04 17:50:09     Applied At                  Migration
2024/02/04 17:50:09     =======================================
2024/02/04 17:50:09     Pending                  -- 00002_1_29_alter_table.sql
```

2. 执行 `up-to 2` 命令进行sql迁移任务

```bash
goose -dir ./sql/migrations/ mysql "user:passwd@tcp(mysql.rongke-base:3306)/your_db?charset=utf8mb4&parseTime=true&loc=Local" up-to 2
2024/02/04 17:50:47 OK   00002_1_29_alter_table.sql (951.85ms)
2024/02/04 17:50:47 goose: successfully migrated database to version: 2
```

3. 再次执行 `status` 命令查看迁移状态

```bash
goose -dir ./sql/migrations/ mysql "user:passwd@tcp(mysql.rongke-base:3306)/your_db?charset=utf8mb4&parseTime=true&loc=Local" status
2024/02/04 17:50:50     Applied At                  Migration
2024/02/04 17:50:50     =======================================
2024/02/04 17:50:50     Sun Feb  4 17:50:45 2024 -- 00002_1_29_alter_table.sql
```

此时所有版本迁移任务都已经执行完毕,可以随时使用 down 命令进行递进降版本,也可以直接使用 down-to 降到指定版本