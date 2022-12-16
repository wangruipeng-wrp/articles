---
title: maven 数据源配置
abbrlink: 243
date: 2021-06-13 15:08:40
description: 记录 maven 数据源配置（spring源、淘宝源）
hide: true
categories:
 - 代码段记录
---

```xml
<mirror>
    <!-- spring源 -->
    <mirror>
        <id>central</id>
        <name>central</name>
        <url>https://repo.spring.io/libs-milestone/</url>
        <mirrorOf>central</mirrorOf>
    </mirror>
    <!-- 淘宝源 -->
    <mirror>
        <id>alimaven</id>
        <name>aliyun maven</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
        <mirrorOf>central</mirrorOf>        
    </mirror>
</mirror>
```