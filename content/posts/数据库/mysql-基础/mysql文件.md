---
title: "Mysql文件"
date: 2022-12-04T23:56:22+08:00
draft: false
categories: ["数据库","mysql"]
---

# 文件类型

+ 参数文件
+ 日志文件
+ Socket 文件
+ pid 文件
+ Mysql 表结构文件
+ 存储引擎文件

# 参数文件

mysql 实例在启动的时候，会查找一些配置参数文件。用来寻找数据库的各种文件所在位置以及指定某些初始化参数。如果没有找到就会找编译Mysql 时指定的默认值和源代码中指定参数的默认值。如果没有找到mysql 架构，还是会启动失败



## 参数

参数就是文本类型的，key/value 形式。

```sql
# 查看数据库中所有参数
show variables 

# like 指定具体的参数key
show variables like ''
```

参数类型

+ 动态
  + Global:修改了Global 当前session 内不会生效，新开的session 会生效，重启mysql 之后，就会恢复到配置文件的参数
  + Session： 修改了Session ，当前Session 立即生效，新Session 不会生效
+ 静态【不可修改】

# 日志文件

+ 错误日志
+ 二进制日志
+ 慢查询日志
+ 查询日志

## 错误日志

错误日志记录mysql 启动、运行、关闭过程。

```sql
show variables like 'log_error';
 /usr/local/mysql/data/mysqld.local.err
```

## 慢查询日志

Mysql 中启动时设置一个阀值，将运行时间大于这个一阀值 的sql 记录到慢查询日志中

```sql
# 设置阀值 单位毫秒
show variables like 'long_query_time';

# 运行sql 是否使用索引 默认关闭
show variables like 'log_queries_not_using_indexes';
```

**mysql 5.1 后，慢查询记录到表中 slow_log**

```sql
# 展示建表语句
show create table mysql.slow_log;

# 查看慢查询输出格式
show variables like 'log_output'; 

set global slow_query_log='ON';-- 启用慢查询 ,加上global，不然会报错的;

# 查看慢sql 表
SELECT * from mysql.slow_log
```

**Mysqldumpslow** 

```text
mysqldumpslow [ OPTS... ] [ LOGS... ]  -- 后跟参数以及log文件的绝对地址;


例：
mysqldumpslow -s c -t 10 /var/run/mysqld/mysqld-slow.log # 取出使用最多的10条慢查询

mysqldumpslow -s t -t 3 /var/run/mysqld/mysqld-slow.log # 取出查询时间最慢的3条慢查询

mysqldumpslow -s t -t 10 -g “left join” /database/mysql/slow-log # 得到按照时间排序的前10条里面含有左连接的查询语句

mysqldumpslow -s r -t 10 -g 'left join' /var/run/mysqld/mysqld-slow.log # 按照扫描行数最多的
```



```sql
CREATE TABLE `slow_log` (
  `start_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  `user_host` mediumtext NOT NULL,
  `query_time` time(6) NOT NULL,
  `lock_time` time(6) NOT NULL,
  `rows_sent` int NOT NULL,
  `rows_examined` int NOT NULL,
  `db` varchar(512) NOT NULL,
  `last_insert_id` int NOT NULL,
  `insert_id` int NOT NULL,
  `server_id` int unsigned NOT NULL,
  `sql_text` mediumblob NOT NULL,
  `thread_id` bigint unsigned NOT NULL
) ENGINE=CSV DEFAULT CHARSET=utf8mb3 COMMENT='Slow log'
```

通过long_query_io 将超过指定逻辑IO 次数的sql 记录到slow log 中，默认是100，

slow_query_type:

+ 0: 不将s q l 记录到slow  log 中
+ 1: 根据运行时间将s q l 记录到slow  log 中
+ 2: 根据逻辑i o 次数将s ql 记录到slow  log 中
+ 3: 根据运行时间、逻辑i o 次数将s ql 记录到slow  log 中

## 查询日志

**查询日志记录了所有对mysql 数据库的请求**，无论这些请求是否正确

mysql 5.1 之后将记录放在general_log 表中

```sql
select * from mysql.general_log;
```

## 二进制文件

**二进制文件记录了对mysql 数据库执行的更新操作，但不包括 select 和show  操作**，若操作没有对数据库产生影响，也有可能会写入二进制文件

**用户想要记录 select 和show  操作了，可以使用查询日志**

二进制文件名称一般为 主机名.二进制序列号：在配置文件中指明 binlog 为文件名，binlog.index为二进制索引文件，用来存储过往生成的二进制日志序列号

二进制文件作用：

+ 恢复
+ 复制：通过复制保证两个数据库同步
+ 审计：根据二进制文件内容来判断是否有对数据库进行注入攻击

参数：

- Max_binglog_size: 单个二进制文件最大值
- bing_cache_size : 有事务的时候，操作会记录到缓存中，提交之后将缓存中的内容写入到二进制文件，【基于会话的】，当数据库事务内容大于bing_cache_size，会写到临时文件中，
- Sync_binlog = [N ] 表示每写缓存多少次就同步到磁盘。默认是0
- Bingo-do-db,binlog-ignore-db 表现需要写入或者忽略的那些库的日志
- binlog_format:记录二进制的格式
  - STATEMENT: 逻辑s ql 语句
  - ROW: 记录表的行更改情况
  - MIXED： 两者结合

binlog_format=ROW可以为数据库恢复和复制更可靠，但会增大文件大小



查看二进制文件：

mysqlbinlog xx0001

## pid 文件

mysql 启动时会将自己的进程ID写入文件中

## innodb 存储引擎文件

### 表空间文件

innodb将数据存储按照表空间的形式存放。默认有一个10MB 的共享表空间，名称为ibdata1 的文件。

可以为每张表设置独立的表空间，命名为: 表名.ibd

```sql
# 查看每张表是否开启独立表空间
show variables like 'innodb_file_per_table';
```

在每张表的表空间中记录了：存储的数据，索引，插入缓冲BITMAP

### 重做日志文件

在innodb 存储引擎目标中有两个文件ib_logfile0，ib_logfile1。是重做日志文件，用于记录每个事务的提交。

每组重做日志文件以循环写入的方式运行，先写第一个文件，写满之后就会写第二个文件，第二个文件写满之后就会重复写第一个文件

+ Innodb_log_file_size: 指定重做日志文件大小
+ Innodb_log_files_in_group: 指定每个组重做日志文件数量
+ 

对比重复日志文件与二进制文件

|          | 二进制文件                          | 重做日志文件                |
| -------- | ----------------------------------- | --------------------------- |
| 写入时机 | 每次提交时写                        | 在事务进行中写入            |
| 内容     | 记录innodb 、MyisAM、Heap引擎的操作 | 只记录innodb 引擎的相关操作 |
| 格式     | 改日志的逻辑操作                    | 每个页的更新的物理操作      |

重做日志文件条目结构

| redo_log_type | space              | page_no    | redo_log_body                          |
| ------------- | ------------------ | ---------- | -------------------------------------- |
| 重做日志类型  | 表空间id(压缩方式) | 页的偏移量 | 重做日志数据部分，需要根据特殊函数修复 |

写入重做日志的时机：

+ master thread 每秒的操作：不论事务是否提交

+ Innodb_flush_log_at_trx_commit参数：

  + 0： 事务提交时不写入，等待主线程操作

  + 1:  提交时同步刷新到磁盘中

    

查看binlog日志

1、先进入到data 目录

2、使用命令查看文件

```mysql
mysqlbinlog --no-defaults --base64-output=decode-rows -v   binlog.000024|more
```

