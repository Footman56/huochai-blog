---
title: "Mysql技巧"
date: 2022-12-04T23:57:25+08:00
draft: false
categories: ["数据库","mysql"]
---

# 1、汉字排序

**默认编码是utf8mb4，所以其实数据库存储的并不是汉字文本。按照内部的存储方式**

```
select * from table order by convert(filed using gbk) asc;
```

# 2、explain 

explain 用于查看sql执行的执行的情况

```
EXPLAIN SELECT * FROM table_name WHERE plan_id ='';
```

| id                  | select_type        | table          | type | possible_keys            | key                         | key_len | ref                             | rows                                          | extra      |
| ------------------- | ------------------ | -------------- | ---- | ------------------------ | --------------------------- | ------- | ------------------------------- | --------------------------------------------- | ---------- |
| SELECT 查询的标识符 | SELECT 查询的类型. | 查询的是哪个表 |      | 此次查询中可能选用的索引 | 此次查询中确切使用到的索引. |         | 哪个字段或常数与 key 一起被使用 | 显示此查询一共扫描了多少行. 这个是一个估计值. | 额外的信息 |

## select_type

- SIMPLE  表示此查询不包含 UNION 查询或子查询
- PRIMARY, 表示此查询是最外层的查询
- UNION, 表示此查询是 UNION 的第二或随后的查询
- UNION RESULT, UNION 的结果
- SUBQUERY, 子查询中的第一个 SELECT

## type

可以根据type 来确定如何查询

- system：表中只有一条数据.

- const: 针对主键或唯一索引的**等值查询扫描**, 最多只返回**一行**数据

- eq_ref: 唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配

  ```
  EXPLAIN SELECT * FROM user_info, order_info WHERE user_info.id = order_info.user_id
  ```

- ref:  针对于**非唯一或非主键索引等值查询**, 或者是使用了 `最左前缀` 规则索引的查询.

  ```
  EXPLAIN SELECT * FROM user_info, order_info WHERE user_info.id = order_info.user_id AND order_info.user_id = 5
  # user_id 是联合索引的第一个索引
  ```

- range： 使用索引范围查询, 通过索引字段范围获取表中部分数据记录. 这个类型通常出现在 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, IN() 操作中.

- index: 表示全索引扫描(full index scan), index 类型则仅仅扫描所有的索引, 而不扫描数据.

  ```
  EXPLAIN SELECT name FROM  user_info;
  # name 是索引
  ```

- ALL: 表示全表扫描

性能：

ALL < index < range < ref < eq_ref < const < system  

## extra

- Using filesort: MySQL 需额外的排序操作,查询 CPœU 资源消耗大.
- Using index: 覆盖索引扫描", 表示查询在索引树中就可查找所需数据
- Using temporary: 查询有使用临时表, 一般出现于排序

# count

在统计数量的时候，也进行排序。

# 查看表锁使用情况

```
# 1.查看当前数据库运行的所有事务（一般查不到，除非发生死锁）
SELECT * FROM information_schema.INNODB_TRX; 

# 杀掉查询结果中锁表的trx_mysql_thread_id 
kill trx_mysql_thread_id


# 查询是否锁表 
show OPEN TABLES where In_use > 0;  
# 查询进程  
show processlist 
# 查询到相对应的进程===然后 kill    id 

 
 # 查看正在锁的事务 
 SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCKS;  
 # 查看等待锁的事务 
 SELECT * FROM INFORMATION_SCHEMA.INNODB_LOCK_WAITS;
 
 # 查看锁执行情况
 show status like 'innodb_row_lock_%';
# Innodb_row_lock_current_waits : 当前等待锁的数量
# Innodb_row_lock_time : 系统启动到现在，锁定的总时间长度
# Innodb_row_lock_time_avg : 每次平均锁定的时间
# Innodb_row_lock_time_max : 最长一次锁定时间
# Innodb_row_lock_waits : 系统启动到现在总共锁定的次数
```

# 查询sql 操作

```
show status like "com_insert%"; -- 获得mysql的插入次数;

show status like "com_delete%"; -- 获得mysql的删除次数;

show status like "com_select%"; -- 获得mysql的查询次数;

show status like "uptime"; -- 获得mysql服务器运行时间

show status like 'connections'; -- 获得mysql连接次数
```

# 函数

## ASCII(s)

返回字符串中第一个字符的ACSII码

```
select ASCII('xaioli');
# 结果： 120
```

## CHAR_LENGTH（s）

返回字符串的长度

```
select CHAR_LENGTH('xiaoli');
# 结果 6
```

## CONCAT(s1,s2,...,sn)

将多个字符串合并

```
select CONCAT('xiaoli','xiaohong');
# xiaolixiaohong
```

CONCAT_WS(x,s1,s2,...,sn)

通过字符X 将多个字符串合并

```
select CONCAT_WS(',','xiaoli','xiaohong');
# xiaoli,xiaohong
```



[**sql函数**]    https://www.runoob.com/mysql/mysql-functions.html Command + click 跳转

​	



# group by



```
使用 union all 来代替 or  查询.

SELECT *
        FROM kpi.kpi_plan_info
        WHERE
        <include refid="select_plan_condition"/>
        <if test="accountId != null and accountId != ''">
            and create_acc_id = #{accountId}
        </if>

        UNION ALL

        SELECT *
        FROM kpi.kpi_plan_info
        WHERE
        <include refid="select_plan_condition"/>
        <if test="accountId != null and accountId != ''">
            and associates_acc_ids LIKE CONCAT('%', #{accountId}, '%')
        </if>
```



mysql操作查询结果case when then else end


--简单Case函数 

CASE column
WHEN <condition> THEN value
WHEN <condition> THEN value
......
ELSE value END

例：
CASE sex 
         WHEN '1' THEN '男' 
         WHEN '2' THEN '女' 
ELSE '其他' END 


--Case搜索函数 
CASE
WHEN <condition> [,<condition>] THEN value
WHEN <condition> [,<condition>] THEN value
......
ELSE value END

例：
CASE WHEN sex = '1' THEN '男' 
         WHEN sex = '2' THEN '女' 
ELSE '其他' END 


（1）已知数据按照另外一种方式进行分组，分析。
SELECT  SUM(population) ,
        CASE country 
                WHEN '中国'     THEN '亚洲' 
                WHEN '印度'     THEN '亚洲' 
                WHEN '日本'     THEN '亚洲' 
                WHEN '美国'     THEN '北美洲' 
                WHEN '加拿大'  THEN '北美洲' 
                WHEN '墨西哥'  THEN '北美洲' 
        ELSE '其他' END  
FROM    country 
GROUP BY CASE country 
                WHEN '中国'     THEN '亚洲' 
                WHEN '印度'     THEN '亚洲' 
                WHEN '日本'     THEN '亚洲' 
                WHEN '美国'     THEN '北美洲' 
                WHEN '加拿大'  THEN '北美洲' 
                WHEN '墨西哥'  THEN '北美洲' 
        ELSE '其他' END; 
        
        

(2) 更新
UPDATE country set population= 
CASE 
	WHEN population > 600 THEN
		population-200
	WHEN population<500 THEN
		population+300
	else population END;
	
建议加上 else ,否则如果条件都不满足的时候，属性会置为null