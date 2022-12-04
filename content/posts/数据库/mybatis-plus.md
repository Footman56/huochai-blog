---
title: "Mybatis Plus"
date: 2022-12-04T23:45:38+08:00
draft: false
categories: ["数据库"]
---

# 零、文档连接

**https://mp.baomidou.com/guide/#%E7%89%B9%E6%80%A7**

# 一、安装

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.3</version>
</dependency>
```

# 二、配置

```java
@SpringBootApplication
@MapperScan("com.baomidou.mybatisplus.samples.quickstart.mapper")
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

