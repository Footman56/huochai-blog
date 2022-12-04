---
title: "Mysql体系"
date: 2022-12-04T23:54:36+08:00
draft: false
categories: ["数据库","mysql"]
---

# 一、定义

```
数据库：物理操作文件或者其他形式文件的集合。文件包括：frm,MYD,MYI,存在二级存储器的文件
实例：后台进程以及共享内存组成。数据库实例才是真正操作文件的。是程序，处于用户与数据库之间
```

**Mysql数据库是单进程多线程**

```
在Mysql数据库启动时会查找配置文件，根据配置文件内容启动程序，如果没有配置文件时候就按照编译时的默认参数来启动

mysql --help|grep my.cnf 指令用于查看启动配置文件的位置

etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf  安装顺序加载配置文件，以最后加载到的配置项为准
```

```
配置文件中有datadir配置项，用于指定数据库文件所在路径（是一个链接）
目录的用户是Mysql、组都是Mysql
```

<img src="/Users/mac/Library/Application Support/typora-user-images/image-20210601000049908.png" alt="image-20210601000049908" style="zoom:50%;" />

<img src="/Users/mac/Library/Application Support/typora-user-images/image-20210601000138493.png" alt="image-20210601000138493" style="zoom:50%;" />

# 二、结构

<img src="/Users/mac/Desktop/mysql.png" alt="mysql" style="zoom:50%;" />

```apl
组成：
	连接池组件
    管理服务和工作组件
    SQL接口组件
    查询分析组件
    优化器组件
    缓存组件
    插件式存储引擎
    物理文件
```

**存储引擎是基于表的，不是基于数据库的**

# 三、InnoDB

```
支持事务，行锁，支持外键，非锁定读（读数据时不会产生锁）
采用聚簇存储，存储主键和行数据
插入缓存，二次写、自适应哈希

```

# 四、Myisam

```
不支持事务、表锁，不支持外键、支持全文索引
缓存池只缓存索引文件，不缓存数据文件
文件以.myd,.myi组成,.myd存储数据、.myi存储索引

```

# 五、连接

```
连接方式有管道、TCP/IP、UNIX套接字、命名管道
```

## 5.1、TCP/IP

```
TCP/IP是常见的网络连接，由客户端向服务器发起连接请求。在请求的时候需要根据权限视图来检查登录情况。
视图中制定了 host（请求地址，%代表所有），user,passwd等
```

## 5.2 UNIX套接字

```
在Linux、UNIX环境中可以通过UNIX套接字来连接
首先查看Unix文件位置： show variables like 'socket';

连接 mysql -udavid -S 文件位置
```

# 六、体系

```
内存块：缓存磁盘上的数据、为线程、进程提供公共的数据结构、重做日志缓存
后台线程：刷新内存池的数据，保证缓存池中的数据是最新的，将已存在的数据写入到磁盘空间
```

<img src="/Users/mac/Library/Application Support/typora-user-images/image-20210607222906047.png" alt="image-20210607222906047" style="zoom:50%;" />

## 1、Master Thread

```
将缓存池中的数据异步刷新到磁盘：脏页的刷新、UNDO页的回收、合并插入缓存
```

## 2、IO Thread

```
InnoDB存储引擎中大量使用AIO(Async IO)来处理Io请求，IO Thread用于处理IO请求的回调
分别是write、read、insert buffer 、log IO Thread  
通过show VARIABLES like 'innodb_version'\G查看线程数


```

## 3、Purge Thread

```
用于回收已经使用并分配的undo页
```

**SHOW ENGINE INNDB STATUS \G： 可以查看存储引擎的信息**

# 七、内存

```
InnoDB是基于磁盘存储的，并将内容按照页的方式进行管理。使用缓存来解决内存与磁盘之间读写的差异。

在数据库进行读取页操作的时候，首先将读取到的页存放在缓冲池中，称为页FIX在缓冲池，下次读取相同的页的时候，
首先判断页在不在缓冲池，如果在就读取缓冲池，不在就读取数据库

数据库中页的修改操作，首先修改缓冲池中的页，然后再以一定的频率刷新到磁盘。
```

```
缓冲池缓存的数据页型有：索引页，数据页，undo页，插入缓冲、自适应哈希索引、InnODB存储的锁信息、数据字典
```

<img src="/Users/mac/Library/Application Support/typora-user-images/image-20210608155040562.png" alt="image-20210608155040562" style="zoom:50%;" />



```
缓冲池中存在三种队列：LRU、Free、Flush.
LRU队列存在查询过的页（有脏页）
Free队列存放未查询过的页
Flush存放脏页（修改过的页）


LRU队列采用谁先被访问谁在队列头的策略。但是有新的页进入队列的时候不是放在队列头，而是放在队列37%的位置（midpoint）[参数是innodb_old_blocks_pct]。

```

```
重做日志缓冲


InnoDB首次将重做日志信息写入到重做日志缓存，之后按照一定频率将其刷新到重做日志文件（每一秒刷新一次）

刷新条件：
Master Thread 每一秒将重做日志缓冲刷新到重做日志文件
每一事务提交时
当重做日志缓冲池剩余空间小于50%
脏页数量太多

[为了避免数据丢失，先将数据写入到重做日志，再写入页中]

在数据库发生宕机之后，只需将Checkpoint 后的重做日志文件进行恢复，不需要完全恢复。
```

**InnoDB存储引擎，使用LSN标记版本 （8字节数字）**

**Checkpoint就是将缓存中的脏页刷回磁盘**

```
有两种刷新方式：
	Sharp Checkpoint: 在数据库关闭时将所有脏页都刷回磁盘
	Fuzzy Checkpoint: 每次只刷新一部分脏页
```

## 插入缓冲

**聚集索引：数据行的物理顺序与列值的顺序是一样的。一张表只有一个聚集索引**

**非聚集索引：数据行的物理顺序与列值的顺序是不一样的**

如果我们查询的值在索引的范围内，就只需做一次查询，如果查询的结果中还有其他的值，就需要再查询所有【二次查询】

```
Insert Buffer 和数据页一样，都是物理页的组成部分
如果采用自增主键的方式，插入会很快

两个条件：
	索引是辅助索引
	索引不唯一
将多个插入合并到一个插入

对Insert、Delete、Update，操作的时候，先写入缓存，在合适的时机将 Insert Buffer合并到索引中。
时机是： 辅助索引页被读取到缓冲池中、Master Thread 、Insert Buffer Bitmap页追踪到改辅助索引页已无可使用空间
```

## 两次写

```
doublewrite有两部分组成：一部分是内存中的doublewirte buffer 大小为2M，另一部分为物理磁盘上共享表空间上连续的128页，大小为2M

第一次写是将被损坏的页写入磁盘
第二次写是将重做日志中失败的内容写入磁盘


skip_innodb_doublewrite 开关来控制是否开启两次写
```

## 自适应哈希（AHI）

```
innodb会自动为查询频繁的数据建立索引。能建立哈希索引带速度提升的时候，会自动创建自适应哈希

自适应哈希适合等值查询，并且要求访问的模式是一样的（查询时，参数出现的位置要固定）
```

## 异步IO

```
Innodb引擎不需要等待一个请求彻底执行完成之后，在返回结果。
并且在查询的时候也可以进行Merge iO操作，将查询连续的页作为一次查询

read ahead 方式的读取是通过AIO完成的，脏页的刷新（磁盘的写入）都是通过AIO实现的
```

## 刷新邻接页

```
在执行脏页刷新的时候，如果邻接的页也是脏页的话，也就merge io ，一起刷新
```

## 启动、关闭、恢复

```
innodb_fast_shutdown 影响在Mysql关闭的时候，Innodb引擎的处理。默认值是1
	0：在数据库关闭的时候，Innodb执行所有full purge and merge insert purge,并将所有脏页刷回磁盘；【在innodb升级的时候，必须调为0】
	1：不执行full purge and merge insert purge，仅执行将所有脏页刷回磁盘
	2：不执行full purge and merge insert purge也不执行将所有脏页刷回磁盘。仅将操作写入日志，在下次启动时进行恢复。


```

**尽量不要使用Kill指令**

