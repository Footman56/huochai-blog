---
title: "Mysql内存"
date: 2022-12-04T23:55:18+08:00
draft: false
categories: ["数据库","mysql"]
---

在Mysql 5.5 之后是默认存储引擎

**是第一个支持ACID 事务的存储引擎**

# 特点

行锁设置、MVCC 、支持外键、提供一致性非锁定读

# 结构

<img src="/Users/peilizhi/Library/Application Support/typora-user-images/image-20211220125057135.png" alt="image-20211220125057135" style="zoom:50%;" />

## 内存池

1. 共多个线程共享数据
2. 缓存磁盘中的文件，，方便快速读取
3. 重做日志缓冲

## 后台线程

1. 刷新缓存池中的数据，保证数据是最近的
2. 将缓存池中的数据存进磁盘

### Master Thread

```
功能：将缓存池中的数据异步刷新到磁盘中
负责：脏页的刷新，合并插入缓存，UNDO 页的回收
```

### IO Thread

InnoDB 使用大量的 AIO 来处理写IO请求

```
包括： write、read 、insert buffer、log IO 
```

```sql
# 查看innodb 版本
show VARIABLES like 'innodb_version';

# 查看innodb 线程
SHOW VARIABLES like '%_threads';
innodb_ddl_threads	4
innodb_log_writer_threads	ON
innodb_parallel_read_threads	4
innodb_purge_threads	4
innodb_read_io_threads	4
innodb_write_io_threads	4
max_delayed_threads	20
max_insert_delayed_threads	20
myisam_repair_threads	1
mysqlx_min_worker_threads	2

# 查看innodb 情况
show ENGINE INnodb STATUS;
```

### Purge Thread

```
mysql 事务提交之后，所使用的undo log 可能就不需要了，Purge thread 会回收已经分配的undo log
```

### Page Cleaner Thread

```
处理脏页刷新操作。
```

## 缓存池

缓存池是介于磁盘和cpu 的缓冲区域，能够有效的提交访问速度

被实现成页面链表

**查询数据先查询缓存池内的数据，没有查到的话就查数据库，之后将数据Fix 进缓存池，之后修改的时候先修改缓存池，并且在合适的时候将数据写进磁盘**

```sql
# 查看缓存池的大小
show VARIABLES like 'INNODB_buffer_pool_size';
```

缓存池中有数据页、索引页、undo页，插入缓存、自适应哈希索引、Innodb存储的锁信息、数据字典信息

内存的结构：

<img src="https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202212042355378.png" alt="image-20211220223014409" style="zoom:50%;" />

## LRU、Free、Flush List

**每一页默认大小为16k**

LRU：最近最少使用策略。将最近读到的页放入到队列的开头，将很久之前读到的数据就在队尾，当队列不能存放新的页时候，清除队尾中的页

InnoDB 使用的是优化的LRU,读到的页并不放在队列开头，而是放在队列的 5/8位置(midpoint )处，并且可以通过参数设置innodb_old_blocks_time用于表示页读到mi d 位置处还需要多久才放入到热端。

这么做是为了避免有一些s q l 操作（扫描整个表）频繁将数据放入到开头，其实这些数据并不是访问次数最多的，只是临时查到了； 

```sql
# 查看midpoint 位置
show variables like 'innodb_old_blocks_pct';

# 查看多少时间放入到热端,值越大，前面的数据越不容易刷出
show variables like 'innodb_old_block_time';
```

数据库刚启动的时候，LRU 队列的是空的，数据都在Free list 中，当需要从缓存中进行分页时候，首先从Free list查看是否有可用的空闲页，有则将该页将Free list 中删除，并加入到LRU List 中，否则根据LRU 算法淘汰末尾的页，将该内存空间分配给新的页

Innodb 支持压缩页技术，将原本的16k 压缩成2k,4k,8k.压缩页组成的队列是 unzip_LRU，unzip_LRU 包含在LRU List 中

在LRU 列表中被修改的页就是脏页，脏页存在Flush list 中

## 重做日志缓存

在内存中还有一部分区域用于 存储重做日志，Innodb 首先将重做日志存入内存中的重做日志缓存，之后再写入重做日志文件。

```sql
# 查看重做日志缓存大小
show variables like 'innodb_log_buffer_size'; 16M
```

重做日志缓存不需要特别大，因为将缓存中的内容刷新到重做日志文件的频率很快

刷新到重做日志的时机：

+ Master Thread 每一秒将重做日志缓存 写入到重做日志文件
+ 事务提交 的时候
+ 重做日志缓存空间中剩余空间小于1/2 时

## checkout point

数据库先写重做日志文件，再写入磁盘。来满足事务的持久性

在数据库宕机的时候，可以将重做日志日志文件写入到磁盘，保证数据不丢失,checkout point 保证写入的数据不会太多，只刷新checkout 之后的操作。

checkout解决：

+ 缩短数据库恢复时间
+ 缓存池不够时，将脏页刷进磁盘
+ 重做日志不可用时，刷新脏页

如果缓存没有空余的空间时，淘汰很久没用的页，如果页是脏页，就强制执行checkout point，写入磁盘

重做日志文件是循环覆盖的，如果被覆盖的内容还是需要使用的，就强制执行checkout point，写入磁盘

Checkout point 可以分为两种： Sharp Checkout(关闭数据库时执行)、Fuzzy Checkout 

执行Fuzzy Checkout 的时机

+ Master thread ：每十秒将脏页以一定的比例写入到磁盘

+ Flush_LRU_List： LRU 要留100页空余可用，不够时，尾部页移除，如果是脏页就Checkout point

+ 重做日志不可用

+ Dirty page too much 

  ```sql
  # 缓存池中脏页的最大占比
  show variables like 'innodb_max_dirty_pages_pct';  90%
  ```

  

## Master Thread

Master Thread 具有最高线程优先级。内部有多个loop ：loop、backgroup loop、flush loop、suspend loop

Loop: 

+ 每一秒做的事
  1. 重做日志缓存刷新到磁盘，尽管事务还没有提交【总是】
  2. 合并插入缓存【可能】：前一秒内IO 操作< 5 
  3. 至多刷新100个脏页到磁盘【可能】 ： 实际脏页数> 最大脏页百分比
  4. 没有活动，切换到backgroud loop【可能】
+ 每十秒做的事
  1. 刷新100个脏页到磁盘【可能】： 10秒内Io 次数<200 
  2. 合并至多5个插入缓存【总是】
  3. 将日志缓存刷新到磁盘【总是】
  4. 删除无用的undo 页【总是】对表的更新、删除操作，原来的行被标记删除，如果后续没有使用，就执行删除
  5. 刷新100个脏页到磁盘【总是】

Backgroup loop:(数据库空闲或则数据库关闭)

+ 删除无用的undo 页【总是】
+ 合并20个插入缓存【总是】
+ 跳回主循环【总是】
+ 不断刷新100页直到满足条件（会跳到flush loop）

flush loop 如果没什么可做的话会进入suspend loop 会将Master Thread 暂时挂起，等待事件发生。

# 关键特性

##  插入缓存

聚集索引：以主键建立的索引，索引的顺序与主键的顺序是一致的。一个表只能有一个聚集索引

非聚集索引：除了主键以外的索引。

Inner buffer 并不存在缓存中，而是实际的物理页。

对于非聚集索引的插入或者更新操作，不是直接插入到索引页，先判断插入到非聚餐索引页是否存在缓冲池，如果在就直接插入，如果不在就先放在insert buffer 对象中，再以一定的频率和情况进行insert buffer 和辅助索引页字节点 merge 操作



使用Insert buffer 条件

+ 索引是辅助索引
+ 索引不唯一

如果在写密集的情况下，insert buffer 会占用过多的缓冲池，可以控制最多占比

## change buffer

InnoDB对 Insert、de lete、update 都做缓冲，分别是；Insert Buffer ,Delete Buffer, Purge Buffer.

Update 分为两步：

1. 原数据标记删除
2. 真正删除原数据

Delete Buffer 对应第一步，Purge Buffer对应第二步

Insert Buffer 、Change Buffer 都是B+ 树

## Merge Insert Buffer

Merge Insert Buffer  就是将插入缓存 合并到实际的缓存中

Merge Insert Buffer 的操作时机：

+ 辅助索引页被读到缓存
+ Insert Buffer Bitmap 页追踪到改辅助索引页已无可用空间

## 两次写

部分写失效：数据库正在写入某页到表中，并且只写了一部分，此时数据库发生宕机。

此时可以通过重做日志文件来恢复，但是重做日志中记录的是对页的物理操作，比如偏移量，如果页已经失效，那么偏移量就没有意义啦。

double wirte 分为两部分，一部分是内存中的double write buffer ，大小是2mb .另一个物理磁盘上共享空间上连续的128页，2MB.

在对脏页进行刷新对时候，通过mempy 将脏页复制到内存中的double write buffer，之后double write buffer 分两次1MB 顺序写到共享空间，之后调用fsync 函数同步磁盘

## 自适应哈希索引

哈希是一种快速的查询方式，时间一般O（1），B+树的查询次数与树的高度有关。

Innodb 会根据查询自动建立哈希索引，需要对页的连续访问模式是一样的，并且只支持等值查询

## 异步io

异步io不会等待i o的执行结果。异步i o可以将多个i o进行合并

## 刷新邻接页

当有脏页需要刷新时，会检查与它相临的页是否是脏页，如果是就一起刷新。

可以将多个i o 合并成一个io

# 启动、关闭、恢复

在关闭mysql 时，innodb_fast_shutdown 影响着innodb 的行为，默认为1 

innodb_fast_shutdown：

+ 0：Innodb完成所有 full purge ,merge insert buffer .将脏页刷新到磁盘。在innodb 升级时必须调成0
+ 1: 不需要 full purge ,merge insert buffer，但是将脏页刷新到磁盘
+ 2: 不需要 full purge ,merge insert buffer，不将脏页刷新到磁盘，将操作写到日志文件中，数据库启动的时候，进行恢复操作



Innodb_force_recovery 影响innodb 恢复情况。默认是0

+ 1: 忽略检查到的corrupt 页
+ 2: 阻止master thread 
+ 3: 不进行事务回滚
+ 4: 不进行插入缓存合并
+ 5: 不查看undo log ,忽略位提交的事务
+ 6: 不进行前滚操作 

**Innodb_force_recovery > 0 ,不能执行 insert、update 、delete**