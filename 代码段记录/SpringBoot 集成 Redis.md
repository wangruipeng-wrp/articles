---
title: Spring Boot 集成 Redis
abbrlink: 44305
date: 2021-09-13 00:21:09
description: 使用 Spring Boot 操作 Redis 其实非常简单，本文主要记录一下整合的步骤，方便后续查看。
hide: true
categories:
 - 代码段记录
---

> 使用 Spring Boot 操作 Redis 其实非常简单，本文主要记录一下整合的步骤，方便后续查看。

<!-- more -->

1. 添加 Redis 依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <version>2.5.4</version>
</dependency>
```

2. 在 yml 文件中配置 Redis 服务器信息
```yml
spring:
  redis:
    database: "指定所使用的是 Redis 中的哪个数据库"
    host: "Redis 服务器 IP 地址"
    port: "Redis 端口号"
    password: "指定登录客户端的密码，如果没有可以不指定"
```

3. 在 Spring Boot 中使用 StringRedisTemplate 操作 Redis 缓存
```java
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

/**
 * Redis 工具类
 */
@Component
@RequiredArgsConstructor(onConstructor = @__(@Autowired))
public class RedisOperator {

    private final StringRedisTemplate redisTemplate;

    /**
     * 以 key 为键获取 Redis 中的值
     */
    public String get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    /**
     * 设置 Redis 键值对
     */
    public void set(String key, String value) {
        set(key, value, -1);
    }

    /**
     * 设置 Redid 键值对
     * @param time 过期时间，单位：秒
     */
    public void set(String key, String value, long time) {
        redisTemplate.opsForValue().set(key, value, time);
    }

    /**
     * 删除 Redis 中的键值对
     */
    public void del(String... keys) {
        for (String key : keys)
            redisTemplate.delete(key);
    }
}
```

> 以上是我在日常开发中真实使用到的一些对 Redis 的操作，在此封装成一个简单的工具类，以后随着使用的越多会封装更多简便的方法。