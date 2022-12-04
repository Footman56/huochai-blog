---
title: "Mysql表"
date: 2022-12-04T23:56:01+08:00
draft: false
categories: ["数据库","mysql"]
---

# 索引组织表

表如果按照主键的顺序存放的话，这个表就叫做索引组织表。

Innodb存储引擎中每个表都有一个索引，如果不显示声明索引的话，会按照下面规则创建

+ 选择创建表时第一个定义的 唯一非空索引
+ 没有 唯一非空索引 时会自动创建一个6字节大小的指针作为索引，（如果有唯一联合索引的时候也会创建6字节大小的指针）

可以通过 _rowid 来查看主键的值，只适合单独的列作为主键，不适合多个列的情况

```sql
select *,_rowid from tablename;
```



# 逻辑结构

**表空间**(Tablespace)  -> **段 **(Segment)-> **区**(Exent)->**页**（page）-> 行（row）

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202212042356161.png" alt="image-20220407001259160" style="zoom:50%;" />

## 表空间

**所有的数据都存在表空间里面**

innodb存储引擎有一个默认的表空间（ibdata1）,默认都放在共享表空间中

如果开启 innodb_file_per_table 属性的话，每一个表都有独立的表空间: table name.idb

```sql
show variables like 'innodb_file_per_table';
```

每张表的表空间里面值存储：数据、索引、插入缓冲Bitmap 页

其他的如 undo信息、插入缓冲索引页、系统事务数据、二次写缓冲 还是存放在共享表空间里面 

```sql
# 未创建表时 idbata1.size = 12582912
# 创建表
CREATE TABLE `table1` (
  `id` int NOT NULL,
  `name` varchar(45) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

创建表时不会增加ibdata1 的容量





## 段

段可以分为：数据段、索引段、回滚段。数据段在B+树的叶子节点，索引段在B+树的非叶子节点

在每个段开始的时候，前32个零散的页用于存放数据，之后每一次申请64个页

## 区

区是由连续的页组成，默认是1M,页可以被压缩

```sql
show variables like 'innodb_page_size';
```

## 页

一个页的大小默认是16KB，可以压缩，设置完成之后不能表中不能再修改页的大小

开启 innodb_file_per_table 属性 之后，每个表的大小是 96KB

页分为：

+ 数据页
+ 索引页
+ undo页
+ 系统页
+ 事务数据页
+ 插入缓冲位图页
+ 插入缓冲空闲列表页
+ 未压缩的二进制大对象页
+ 压缩的二进制大对象页（compressed BLOB Page）