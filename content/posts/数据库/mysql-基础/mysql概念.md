---
title: "Mysql概念"
date: 2022-12-04T23:55:00+08:00
draft: false
categories: ["数据库","mysql"]
---

数据库：各种存储数据的文件集合

实例：访问数据库文件的程序

Mysql数据库是单进程多线程。**数据库的实例就在系统上表现为一个进程**

在启动Mysql 的时候会根据配置文件来启动，**如果没有找到配置文件的话，就按照编译时的默认参数来启动数据库**，Oracle 则会提示失败。

```sh
mysql --help |grep my.cnf

/etc/my.cnf /etc/mysql/my.cnf /usr/local/mysql/etc/my.cnf ~/.my.cnf
```

**mysql 启动时会以最后一个配置文件中定义的参数为准。**

```mysql
# 查看数据存储的位置： 
show variables like 'datadir';

查看权限为
drwxr-x---  35 _mysql  _mysql    1120 12 18 18:16 data
```

**存储引擎是基于表的，不是基于数据库的**

体系结构

![2019110514034932](/Users/peilizhi/Pictures/2019110514034932.png)



在创建表的时候可以指定 存储引擎

```sql
create table table_name engine=InnoDB
```

**链接数据库的时候本质是 进程通讯，使用TCP/IP**

```sql
# 指定服务器、用户、密码
mysql -h ip -u root -p 
```

在链接的时候需要查看权限视图，检查用户是否可以登陆（可以登陆的ip，密码）

```sql
use mysql;
select host,user,password form user;
```



**show variables like '' : 可以查看一些配置项**





innodb 架构图：

![点击查看源网页](https://gcore.jsdelivr.net/gh/Footman56/imageBeds/202212042355614.com&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg)