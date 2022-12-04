---
title: "Mysql事务"
date: 2022-12-04T23:56:46+08:00
draft: false
categories: ["数据库","mysql"]
---



# 事务

# 隔离级别

# 事务实现

## 1、锁

## 共享锁（share lock）

共享锁也叫读锁，可以允许多个事务读取，但不支持其它事务进行更新操作

允许持有锁的事务读取一行

**在sql 中手动加锁: select * from table where 1 lock in share mode**

```sql
begin;
SELECT * FROM student WHERE id =1 lock in share  MODE;


begin;
SELECT * FROM student WHERE id =2 lock in share MODE; # 执行ok ,多个事务可以共享读锁
```



## 排他锁(exclusive lock)

写锁，只要一个事务获取到这一行的排他锁之后，其他事务就不能获取这一行的共享锁和排他锁

允许持有排他锁的事务更新或者删除一行

**新增、修改、删除都会默认加上排他锁**

**在sql 中手动加锁： select * from table where 1 for update**



**共享锁和排他锁都是行锁**

```sql
BEGIN;
UPDATE student SET student_name='xxxx' WHERE id=2;
COMMIT

SELECT * FROM student WHERE id =2 for UPDATE;
```

**会针对每一行进行上锁**

事务一持有S 锁【针对同一行】：

+ 事务二可以获取s锁，不可以获取X 锁

事务一持有X 锁【针对同一行】：

+ 事务二不可以获取s锁，不可以获取X 锁



**当innodb 搜索或扫描一个表索引时，它会在遇到的索引记录上设置共享或排他锁。因此，行级锁实际上是索引记录锁。**

如果表没有创建索引的话，innodb会创建一个默认索引。

## 意向锁

意向锁是表锁，是为了快速判断表是否可以加锁，否则的话需要逐行去判断行上是否有事务。

**意向锁指示事务稍后需要表中的一行使用哪种类型的锁(共享或排他)。**

给行加上 共享锁或者排他锁之后，表会自动加上 共享意向锁或者排他意向锁：为了提高表锁效率

```sql
给表加锁
lock tables table_name write;
lock tables table_name read;
```

 intention shared lock (IS) (`IS`) ：指示事务稍后会对表中的每一行加上S Lock

 intention exclusive lock (IX) (`IS`) ：指示事务稍后会对表中的每一行加上X Lock

```sql
# 为表加上 IS
select ... for share;

# 为表加上 IX
select ... for update;
```

**事务在获取表中某一行的共享锁之前，必须首先获取表上的IS锁或更强的锁。**

**在事务可以在表中的行中获取独占锁之前，必须首先在表格上获取IX锁。**



|      | X    | S    | IX                     | IS   |
| ---- | ---- | ---- | ---------------------- | ---- |
| X    | N    | N    | N                      | N    |
| S    | N    | Y    | N(意味着后面会加入X锁) | Y    |
| IX   | N    | N    | Y                      | Y    |
| IS   | N    | Y    | Y                      | Y    |



## 记录锁(Record lock)

每一条记录，使用等值查询的时候，每一条匹配到的就是上锁。粒度最小.

可以防止对同一行进行 删除、更新。

**记录锁总是锁定索引记录，即使表没有定义索引。对于这种情况，InnoDB会创建一个隐藏的聚集索引，并使用这个索引进行记录锁定。**



## 间隙锁(gap lock)

不同索引之间的间隙

查询的记录不存在，情况是查询语句中有范围，

间隙锁主要是阻塞 insert,相同的间隙锁之间不冲突

**gap lock 可以有效避免幻读**

**间隙锁是索引记录之间的间隙上的锁，或者第一个索引记录之前或最后一个索引记录之后的间隙上的锁。**【开区间】

间隙锁是性能和并发性之间权衡的一部分，



**如果查询条件使用唯一索引的话，gap lock 会退化成 Record lock**

**如果id没有被索引或具有非唯一索引，则该语句将锁定前面的间隙。**

冲突锁可以由不同的事务在间隙上持有。例如，事务A可以持有一个间隙上的共享间隙锁(间隙s锁)，而事务B持有同一个间隙上的排他间隙锁(间隙x锁)。

**相同的间隙可以在不同的事务中持有不同的间隙锁（S或者X 都可以）**

## 临键锁(Next-key  lock)

左开右闭区见。**临键锁=记录锁+间隙锁**

外键检查约束以及重复键检查会使用间隙锁封锁区间

**next-key锁是索引记录锁加上索引记录之前的间隙锁。**

如果一个会话对索引中的记录R有一个共享或排他锁，那么另一个会话不能在紧挨着索引顺序的R**之前**的间隙插入一个新的索引记录。



