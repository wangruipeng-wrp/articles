---
title: Spring Boot 跨域处理
abbrlink: 63535
date: 2021-06-13 15:12:01
description: 记录 Spring Boot 中跨域配置的代码。
hide: true
categories:
 - 代码段记录
---

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;
import org.springframework.web.filter.CorsFilter;

@Configuration
public class CorsConfig {

    @Bean
    public CorsFilter corsFilter() {
        // 1. 添加cors配置信息
        CorsConfiguration config = new CorsConfiguration();

        // 设置允许请求的域名
        config.addAllowedOrigin("https://www.wrp.cool");

        // 设置是否发送cookie信息
        config.setAllowCredentials(true);

        // 设置允许请求的方式
        config.addAllowedMethod("*");

        // 设置允许的header
        config.addAllowedHeader("*");

        // 2. 为url添加映射路径
        UrlBasedCorsConfigurationSource corsSource = new UrlBasedCorsConfigurationSource();
        corsSource.registerCorsConfiguration("/**", config);

        // 3. 返回重新定义好的corsSource
        return new CorsFilter(corsSource);
    }
}
```
