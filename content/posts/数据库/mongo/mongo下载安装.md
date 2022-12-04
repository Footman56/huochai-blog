---
title: "Mongo下载安装"
date: 2022-12-04T23:52:54+08:00
draft: false
categories: ["数据库","mongo"]
---

# 一、使用Brew 安装

```
brew tap mongodb/brew
// 下载 后面指定版本
brew install mongodb
```

<img src="/Users/peilizhi/screenshots/截屏2021-08-13 17.41.49.png" alt="截屏2021-08-13 17.41.49" style="zoom:50%;" />

执行 echo 'export PATH="/usr/local/opt/mongodb-community@4.4/bin:$PATH"' >> ~/.zshrc

# 二、启动

```
brew services start mongodb
```

如果想要使用 mongod 来启动并制定配置文件位置

在`.bash_profile`添加配置

```
# MongoDB Aliases
alias mongod="mongod --config /usr/local/etc/mongod.conf --fork"
```

启动问题

```
如果启动时提示：Data directory /data/db not found. Create the missing directory or specify another path using (1) the --dbpath command line option, or (2) by adding the 'storage.dbPath' option in the configuration file."}}
在配置文件中指定了新的data位置。
```

# 三、配置文件

```yaml
systemLog:
  destination: file
  path: /usr/local/var/log/mongodb/mongo.log
  logAppend: true
storage:
  dbPath: /usr/local/var/mongodb
net:
  bindIp: 127.0.0.1
```

# 四、创建员工

## 4.0 启动mongo

## 4.1 已非授权模式登陆

```sh
mongo
```

## 4.2 切换到admin数据库

```sh
use admin
```

## 4.3 创建用户

```sh
db.createUser({
 user: "huochai",
 pwd: "Asdf1234",
 roles:[ "dbOwner" ]
 })
```

## 4.4 认证

```sh
db.auth("huochai","Asdf1234")
```

## 4.5 登陆客户端

url mongodb://huochai:Asdf1234@127.0.01:27017



数据库之间的账号是独立的，需要在新建的数据库中在重新创建用户

mongodb://appraisal:appraisal@101.200.156.46:27018/appraisal?authSource=appraisal





