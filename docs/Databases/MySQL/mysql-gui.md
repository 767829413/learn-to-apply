# Mysql 应用指南

## SQL 执行过程

学习 Mysql，最好是先从宏观上了解 Mysql 工作原理。

> 参考：[Mysql 工作流](https://github.com/767829413/learn-to-apply/blob/main/docs/Databases/MySQL/mysql-workflow.md)

## 存储引擎

在文件系统中，Mysql 将每个数据库（也可以成为 schema）保存为数据目录下的一个子目录。创建表示，Mysql 会在数据库子目录下创建一个和表同名的 `.frm` 文件保存表的定义。因为 Mysql 使用文件系统的目录和文件来保存数据库和表的定义，大小写敏感性和具体平台密切相关。Windows 中大小写不敏感；类 Unix 中大小写敏感。**不同的存储引擎保存数据和索引的方式是不同的，但表的定义则是在 Mysql 服务层统一处理的。**

### 选择存储引擎

#### Mysql 内置的存储引擎

```shell
mysql> SHOW ENGINES;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.00 sec)
```

- **InnoDB** - Mysql 的默认事务型存储引擎，并且提供了行级锁和外键的约束。性能不错且支持自动崩溃恢复。
- **MyISAM** - Mysql 5.1 版本前的默认存储引擎。特性丰富但不支持事务，也不支持行级锁和外键，也没有崩溃恢复功能。
- **CSV** - 可以将 CSV 文件作为 Mysql 的表来处理，但这种表不支持索引。
- **Memory** - 适合快速访问数据，且数据不会被修改，重启丢失也没有关系。
- **NDB** - 用于 Mysql 集群场景。

#### 如何选择合适的存储引擎

大多数情况下，InnoDB 都是正确的选择，除非需要用到 InnoDB 不具备的特性。

如果应用需要选择 InnoDB 以外的存储引擎，可以考虑以下因素：

- 事务：如果需要支持事务，InnoDB 是首选。如果不需要支持事务，且主要是 SELECT 和 INSERT 操作，MyISAM 是不错的选择。所以，如果 Mysql 部署方式为主备模式，并进行读写分离。那么可以这么做：主节点只支持写操作，默认引擎为 InnoDB；备节点只支持读操作，默认引擎为 MyISAM。
- 并发：MyISAM 只支持表级锁，而 InnoDB 还支持行级锁。所以，InnoDB 并发性能更高。
- 外键：InnoDB 支持外键。
- 备份：InnoDB 支持在线热备份。
- 崩溃恢复：MyISAM 崩溃后发生损坏的概率比 InnoDB 高很多，而且恢复的速度也更慢。
- 其它特性：MyISAM 支持压缩表和空间数据索引。

#### 转换表的存储引擎

下面的语句可以将 mytable 表的引擎修改为 InnoDB

```sql
ALTER TABLE mytable ENGINE = InnoDB
```

### MyISAM

MyISAM 设计简单，数据以紧密格式存储。对于只读数据，或者表比较小、可以容忍修复操作，则依然可以使用 MyISAM。

MyISAM 引擎使用 B+Tree 作为索引结构，**叶节点的 data 域存放的是数据记录的地址**。

MyISAM 提供了大量的特性，包括：全文索引、压缩表、空间函数等。但是，MyISAM 不支持事务和行级锁。并且 MyISAM 不支持崩溃后的安全恢复。

### InnoDB

InnoDB 是 MySQL 默认的事务型存储引擎，只有在需要 InnoDB 不支持的特性时，才考虑使用其它存储引擎。

然 InnoDB 也使用 B+Tree 作为索引结构，但具体实现方式却与 MyISAM 截然不同。MyISAM 索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。而**在 InnoDB 中，表数据文件本身就是按 B+Tree 组织的一个索引结构**，这棵树的叶节点 data 域保存了完整的数据记录。这个**索引的 key 是数据表的主键**，因此**InnoDB 表数据文件本身就是主索引**。

InnoDB 采用 MVCC 来支持高并发，并且实现了四个标准的隔离级别。其默认级别是可重复读（REPEATABLE READ），并且通过间隙锁（next-key locking）防止幻读。

InnoDB 是基于聚簇索引建立的，与其他存储引擎有很大不同。在索引中保存了数据，从而避免直接读取磁盘，因此对查询性能有很大的提升。

内部做了很多优化，包括从磁盘读取数据时采用的可预测性读、能够加快读操作并且自动创建的自适应哈希索引、能够加速插入操作的插入缓冲区等。

支持真正的在线热备份。其它存储引擎不支持在线热备份，要获取一致性视图需要停止对所有表的写入，而在读写混合场景中，停止写入可能也意味着停止读取。

## 数据类型

### 整型

`TINYINT`, `SMALLINT`, `MEDIUMINT`, `INT`, `BIGINT` 分别使用 `8`, `16`, `24`, `32`, `64` 位存储空间，一般情况下越小的列越好。

**`UNSIGNED` 表示不允许负值，大致可以使正数的上限提高一倍**。

`INT(11)` 中的数字只是规定了交互工具显示字符的个数，对于存储和计算来说是没有意义的。

### 浮点型

`FLOAT` 和 `DOUBLE` 为浮点类型。

`DECIMAL` 类型主要用于精确计算，代价较高，应该尽量只在对小数进行精确计算时才使用 `DECIMAL` ——例如存储财务数据。数据量比较大的时候，可以使用 `BIGINT` 代替 `DECIMAL`。

`FLOAT`、`DOUBLE` 和 `DECIMAL` 都可以指定列宽，例如 `DECIMAL(18, 9)` 表示总共 18 位，取 9 位存储小数部分，剩下 9 位存储整数部分。

### 字符串

主要有 `CHAR` 和 `VARCHAR` 两种类型，一种是定长的，一种是变长的。

**`VARCHAR` 这种变长类型能够节省空间，因为只需要存储必要的内容。但是在执行 UPDATE 时可能会使行变得比原来长**。当超出一个页所能容纳的大小时，就要执行额外的操作。MyISAM 会将行拆成不同的片段存储，而 InnoDB 则需要分裂页来使行放进页内。

`VARCHAR` 会保留字符串末尾的空格，而 `CHAR` 会删除。

### 时间和日期

MySQL 提供了两种相似的日期时间类型：`DATATIME` 和 `TIMESTAMP`。

#### DATATIME

能够保存从 1001 年到 9999 年的日期和时间，精度为秒，使用 8 字节的存储空间。

它与时区无关。

默认情况下，MySQL 以一种可排序的、无歧义的格式显示 DATATIME 值，例如“2008-01-16 22:37:08”，这是 ANSI 标准定义的日期和时间表示方法。

#### TIMESTAMP

和 UNIX 时间戳相同，保存从 1970 年 1 月 1 日午夜（格林威治时间）以来的秒数，使用 4 个字节，只能表示从 1970 年 到 2038 年。

它和时区有关，也就是说一个时间戳在不同的时区所代表的具体时间是不同的。

MySQL 提供了 FROM_UNIXTIME() 函数把 UNIX 时间戳转换为日期，并提供了 UNIX_TIMESTAMP() 函数把日期转换为 UNIX 时间戳。

默认情况下，如果插入时没有指定 TIMESTAMP 列的值，会将这个值设置为当前时间。

应该尽量使用 TIMESTAMP，因为它比 DATETIME 空间效率更高。

### BLOB 和 TEXT

`BLOB` 和 `TEXT` 都是为了存储大的数据而设计，前者存储二进制数据，后者存储字符串数据。

不能对 `BLOB` 和 `TEXT` 类型的全部内容进行排序、索引。

### 枚举类型

大多数情况下没有使用枚举类型的必要，其中一个缺点是：枚举的字符串列表是固定的，添加和删除字符串（枚举选项）必须使用`ALTER TABLE`（如果只只是在列表末尾追加元素，不需要重建表）。

### 类型的选择

- 整数类型通常是标识列最好的选择，因为它们很快并且可以使用 `AUTO_INCREMENT`。

- `ENUM` 和 `SET` 类型通常是一个糟糕的选择，应尽量避免。
- 应该尽量避免用字符串类型作为标识列，因为它们很消耗空间，并且通常比数字类型慢。对于 `MD5`、`SHA`、`UUID` 这类随机字符串，由于比较随机，所以可能分布在很大的空间内，导致 `INSERT` 以及一些 `SELECT` 语句变得很慢。
  - 如果存储 UUID ，应该移除 `-` 符号；更好的做法是，用 `UNHEX()` 函数转换 UUID 值为 16 字节的数字，并存储在一个 `BINARY(16)` 的列中，检索时，可以通过 `HEX()` 函数来格式化为 16 进制格式。

## 索引

> 详见：[Mysql 索引](https://github.com/767829413/learn-to-apply/blob/main/docs/Databases/MySQL/mysql-index.md)

## 锁

> 详见：[Mysql 锁](https://github.com/767829413/learn-to-apply/blob/main/docs/Databases/MySQL/mysql-lock.md)

## 事务

> 详见：[Mysql 事务](https://github.com/767829413/learn-to-apply/blob/main/docs/Databases/MySQL/mysql-transaction.md)

## 性能优化

> 详见：[Mysql 性能优化](https://github.com/767829413/learn-to-apply/blob/main/docs/Databases/MySQL/mysql-improvement.md)

## 复制

### 主从复制

Mysql 支持两种复制：基于行的复制和基于语句的复制。

这两种方式都是在主库上记录二进制日志，然后在从库重放日志的方式来实现异步的数据复制。这意味着：复制过程存在时延，这段时间内，主从数据可能不一致。

主要涉及三个线程：binlog 线程、I/O 线程和 SQL 线程。

- **binlog 线程** ：负责将主服务器上的数据更改写入二进制文件（binlog）中。
- **I/O 线程** ：负责从主服务器上读取二进制日志文件，并写入从服务器的中继日志中。
- **SQL 线程** ：负责读取中继日志并重放其中的 SQL 语句。

<div align="center">
<img src="https://pic.imgdb.cn/item/65d58f3e9f345e8d03e33ca7.png" />
</div>

### 读写分离

主服务器用来处理写操作以及实时性要求比较高的读操作，而从服务器用来处理读操作。

读写分离常用代理方式来实现，代理服务器接收应用层传来的读写请求，然后决定转发到哪个服务器。

MySQL 读写分离能提高性能的原因在于：

- 主从服务器负责各自的读和写，极大程度缓解了锁的争用；
- 从服务器可以配置 MyISAM 引擎，提升查询性能以及节约系统开销；
- 增加冗余，提高可用性。

<div align="center">
<img src="https://pic.imgdb.cn/item/65d58f639f345e8d03e3b7de.png" />
</div>

## 参考资料

- [《高性能 MySQL》](https://book.douban.com/subject/23008813/)
- [20+ 条 MySQL 性能优化的最佳经验](https://coolshell.cn/articles/1846.html)
- [How to create unique row ID in sharded databases?](https://stackoverflow.com/questions/788829/how-to-create-unique-row-id-in-sharded-databases)
- [SQL Azure Federation – Introduction](http://geekswithblogs.net/shaunxu/archive/2012/01/07/sql-azure-federation-ndash-introduction.aspx)