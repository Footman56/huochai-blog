---
title: "Mysql死锁"
date: 2022-12-04T23:58:17+08:00
draft: false
categories: ["数据库","mysql"]
---



# 解读 SHOW ENGINE INNODB STATUS;

**显示的不是当前状态，是过去一段时间范围内innoDB 执行情况**

| BACKGROUND THREAD                     | 后台Master线程                                               |
| :------------------------------------ | :----------------------------------------------------------- |
| SEMAPHORES                            | 信号量信息                                                   |
| LATEST DETECTED DEADLOCK              | 最近一次死锁信息，只有产生过死锁才会有                       |
| TRANSACTIONS                          | 事物信息                                                     |
| FILE I/O                              | IO Thread信息                                                |
| INSERT BUFFER AND ADAPTIVE HASH INDEX | INSERT BUFFER和自适应HASH索引                                |
| LOG                                   | 日志                                                         |
| BUFFER POOL AND MEMORY                | BUFFER POOL和内存                                            |
| INDIVIDUAL BUFFER POOL INFO           | 如果设置了多个BUFFER POOL实例，这里显示每个BUFFER POOL信息。可通过innodb_buffer_pool_instances参数设置 |
| ROW OPERATIONS‍‍                        | 行操作统计信息‍‍                                               |
| END OF INNODB MONITOR OUTPU           | 输出结束语                                                   |

```sql
# 当前日期
=====================================
2021-12-23 18:53:51 7f0c9cd28700 INNODB MONITOR OUTPUT
=====================================
# 自上次输出以来的时间
Per second averages calculated from the last 39 seconds


-----------------
BACKGROUND THREAD
-----------------
# master Thread 执行情况
# srv_active为之前的每秒的循环，srv_idle为每10秒的的循环；srv_shutdown为停止的循环，通常为0，只在MySQL关闭时才会增加。
# 根据循环次数可大概判断当前数据库负载情况。如果每秒循环次数少，每10秒次数多，证明当前负载很低；如果每秒循环次数多，每10秒次数少，远大于10：1，证明当前负载很高。
srv_master_thread loops: 4346148 srv_active,  0 srv_shutdown, 2170504 srv_idle
srv_master_thread log flush and writes: 6516648



----------
高并发相关
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 7814572
OS WAIT ARRAY INFO: signal count 12055502
Mutex spin waits 67497239, rounds 115662293, OS waits 610936
RW-shared spins 20467070, rounds 416284062, OS waits 6686750
RW-excl spins 1874510, rounds 19984783, OS waits 189941
Spin rounds per wait: 1.71 mutex, 20.34 RW-shared, 10.66 RW-excl


------------------------
上次死锁情况
LATEST DETECTED DEADLOCK
------------------------
2021-12-22 20:11:40 7f0bf43d2700
# 第一个事务
*** (1) TRANSACTION:
# ACTIVE 表示事务活动时间 ,inserting:事务当前运行状态，其他状态：fetching rows，updating，deleting，inserting
TRANSACTION 1191064881, ACTIVE 24 sec inserting
# 有几张表被使用，是否有表锁，有几个表锁
mysql tables in use 1, locked 1
# 事务正在等待锁，锁链表结构的长度，每个结构表示持有的锁。为事务分配的堆内存大小，多少个行锁，有多少 undo 操作
LOCK WAIT 4 lock struct(s), heap size 1184, 4 row lock(s), undo log entries 2
MySQL thread id 8000428, OS thread handle 0x7f0bf6cf4700, query id 1632526539 218.106.117.210 saas update
# 正在等待锁的sql语句,有可能不是罪魁祸首
insert into account values(null,'Jay',100)
# 正在等待的锁类型
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
# 锁类型，要加锁的位置
RECORD LOCKS space id 8985 page no 4 n bits 72 index `idx_name` of table `logs`.`account` trx id 1191064881 
lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 3; hex 576569; asc Wei;;
 1: len 4; hex 80000002; asc     ;;


# 第二个事务
*** (2) TRANSACTION:
# 执行的操作类型
TRANSACTION 1191068252, ACTIVE 13 sec inserting
# 当前事务使用1个表，持有1个锁
mysql tables in use 1, locked 1
# LOCK WAIT表示正在等待锁, 6 lock struct(s) = 锁链表的长度为6 每个链表节点代表该事务持有的一个锁结构；heap size=表示事务分配的锁堆内存大小；5 row lock(s) = 该事物持有锁的个数
5 lock struct(s), heap size 1184, 4 row lock(s), undo log entries 2
MySQL thread id 8000380, OS thread handle 0x7f0bf43d2700, query id 1632530059 218.106.117.210 saas update
# 执行的操作类型
insert into account values(null,'Yan',100)
*** (2) HOLDS THE LOCK(S):
# 锁住的资源 index：加锁的索引 table：锁定记录的表，
# page no =事务锁定页的数量，若是表锁，该值为null
RECORD LOCKS space id 8985 page no 4 n bits 72 index `idx_name` of table `logs`.`account` trx id 1191068252 lock_mode X locks gap before rec
Record lock, heap no 4 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 3; hex 576569; asc Wei;;
 1: len 4; hex 80000002; asc     ;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 8985 page no 4 n bits 72 index `idx_name` of table `logs`.`account` trx id 1191068252 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

# 回归的事务
*** WE ROLL BACK TRANSACTION (1)

# 事务信息
------------
TRANSACTIONS
------------
Trx id counter 1214468655
Purge done for trx's n:o < 1214468577 undo n:o < 0 state: running but idle
History list length 1220
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 0, not started
MySQL thread id 8106341, OS thread handle 0x7f0c9cd28700, query id 1678904121 11.192.101.45 saas init
/* Query from DMS-WEBSQL-0-Qid_1640256830846 by user 279411300834433704 */ SHOW ENGINE INNODB STATUS
---TRANSACTION 1214467717, not started
MySQL thread id 8106333, OS thread handle 0x7f0bf43d2700, query id 1678902294 10.31.145.66 saas
---TRANSACTION 1214467397, not started
MySQL thread id 8106335, OS thread handle 0x7f0b8d3d1700, query id 1678901639 10.31.151.204 saas
---TRANSACTION 1214467230, not started
MySQL thread id 8106216, OS thread handle 0x7f0c049ea700, query id 1678901353 10.30.45.75 saas、
---TRANSACTION 1210609693, not started
MySQL thread id 3870812, OS thread handle 0x7f0c170c3700, query id 1678901177 10.31.145.188 saas
---TRANSACTION 1094399488, not started
MySQL thread id 2697450, OS thread handle 0x7f0c1ffff700, query id 1678894754 192.168.16.1 saas


--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
Pending normal aio reads: 0 [0, 0, 0, 0] , aio writes: 0 [0, 0, 0, 0] ,
 ibuf aio reads: 0, log i/o's: 0, sync i/o's: 0
Pending flushes (fsync) log: 0; buffer pool: 0
226529896 OS file reads, 58068025 OS file writes, 28146659 OS fsyncs
23.23 reads/s, 16384 avg bytes/read, 13.36 writes/s, 5.62 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 3079, seg size 3081, 794285 merges
merged operations:
 insert 1444947, delete mark 120928, delete 46386
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 276671, node heap has 264 buffer(s)
5353.43 hash searches/s, 1740.62 non-hash searches/s
---
LOG
---
Log sequence number 59275700043
Log flushed up to   59275700025
Pages flushed up to 59275693466
Last checkpoint at  59275693466
0 pending log writes, 0 pending chkp writes
22166060 log i/o's done, 4.59 log i/o's/second

# buffer pool 信息
----------------------
BUFFER POOL AND MEMORY
----------------------
# 分配给缓冲池的总内存(以字节为单位)。
Total memory allocated 137363456; in additional pool allocated 0
# 分配给InnoDB数据字典的总内存(以字节为单位)。
Dictionary memory allocated 10573223
# 分配给缓冲池的页面的总大小。
Buffer pool size   8191
# 缓冲池空闲列表的总页大小。
Free buffers       1022
# buffer pool LRU列表的总页面大小。
Database pages     6905
# buffer pool old LRU子列表的总页面大小。
Old database pages 2528
Modified db pages  32
Pending reads 0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 94310767, not young 8050267524
10.82 youngs/s, 836.34 non-youngs/s
Pages read 226502422, created 614802, written 34141947
23.23 reads/s, 0.10 creates/s, 8.56 writes/s
Buffer pool hit rate 999 / 1000, young-making rate 0 / 1000 not 38 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 6905, unzip_LRU len: 0
I/O sum[2140]:cur[2], unzip sum[0]:cur[0]



--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Main thread process no. 1, id 139692453914368, state: sleeping
Number of rows inserted 5027305, updated 15495535, deleted 381933, read 64931462045
0.64 inserts/s, 2.31 updates/s, 0.46 deletes/s, 7336.97 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================

```

