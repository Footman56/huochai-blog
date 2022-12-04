---
title: "Mybatis使用"
date: 2022-12-04T23:51:40+08:00
draft: false
categories: ["数据库","mybatis"]
---

# 一、 安装

```xml
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
  <version>x.x.x</version>
</dependency>
```



# 二、简单使用

**每个基于 MyBatis 的应用都是以一个 SqlSessionFactory 的实例为核心的。SqlSessionFactory 的实例可以通过 SqlSessionFactoryBuilder 获得。而 SqlSessionFactoryBuilder 则可以从 XML 配置文件或一个预先配置的 Configuration 实例来构建出 SqlSessionFactory 实例。**

## 2.1 如何创建SqlSessionFactory

答：采用工厂方法，先创建SqlSessionFactoryBuilder，之后再构建SqlSessionFactory。

# 2.2 如果创建SqlSessionFactoryBuilder

答：xml 配置文件或者java配置类

### 2.2.1 xml配置文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>
  
  
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
  
  
  <mappers>
    <mapper resource="org/mybatis/example/BlogMapper.xml"/>
  </mappers>
  
  
</configuration>
```

```
可以将xml理解成java对象，里面有environments、mappers等属性
```

```java
// 通过读取文件来创建
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```



XML 配置文件中包含了对 MyBatis 系统的核心设置，包括获取数据库连接实例的数据源（DataSource）以及决定事务作用域和控制方式的事务管理器（TransactionManager）。

### 2.2.2 java配置类

```java
DataSource dataSource = BlogDataSourceFactory.getBlogDataSource();
TransactionFactory transactionFactory = new JdbcTransactionFactory();

// 创建环境
Environment environment = new Environment("development", transactionFactory, dataSource);
// 构造配置
Configuration configuration = new Configuration(environment);

configuration.addMapper(BlogMapper.class);

// 创建SqlSessionFactory，根据配置对象创建
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(configuration);
```

## 2.3 如果使用SqlSessionFactory

答：创建SqlSession。大多数工厂类的作用

## 2.4 SqlSession作用

答：SqlSession 提供了在数据库执行 SQL 命令所需的所有方法。你可以通过 SqlSession 实例来直接执行已映射的 SQL 语句。【是执行已映射的SQL语句】

```java
// 获取SqlSession
try (SqlSession session = sqlSessionFactory.openSession()) {
  // 使用 指定语句的参数和返回值相匹配的接口（比如 BlogMapper.class）
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  // 执行已映射的SQL
  Blog blog = mapper.selectBlog(101);
}
```

## 2.5 如何映射SQL语句

答：可以使用XML定义或者注解定义

### 2.5.1 XML定义

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!---定义命名空间--->
<mapper namespace="org.mybatis.example.BlogMapper">
  <select id="selectBlog" resultType="Blog">
    select * from Blog where id = #{id}
  </select>
</mapper>
```

它在命名空间 “org.mybatis.example.BlogMapper” 中定义了一个名为 “selectBlog” 的映射语句

该命名就可以直接映射到在命名空间中同名的映射器类，并将已映射的 select 语句匹配到对应名称、参数和返回类型的方法。

就是建立xml文件与java映射器类 之间的关系。通常命名空间可以指定映射器类的全类型



```
命名空间的作用有两个，一个是利用更长的全限定名来将不同的语句隔离开来，同时也实现了你上面见到的接口绑定。
	一个映射器类就是一个sql语句的小单元
```

### 2.5.2 注解定义

```java
// 同命名空间
package org.mybatis.example;
public interface BlogMapper {
  @Select("SELECT * FROM blog WHERE id = #{id}")
  Blog selectBlog(int id);
}
```

